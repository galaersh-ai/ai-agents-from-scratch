# Объяснение кода: `tool-routing-embeddings.js`

Этот пример загружает **две** GGUF-модели через один экземпляр `getLlama()`: **chat-модель** (Qwen3) и **специализированную модель эмбеддингов** (bge-small-en). Только модель эмбеддингов использует `createEmbeddingContext()`.

## Запуск

```bash
node examples/15_tool-routing-embeddings/tool-routing-embeddings.js
```

Требуются оба файла в `models/` (см. [DOWNLOAD.md](../../DOWNLOAD.md)): `Qwen3-1.7B-Q8_0.gguf` и `bge-small-en-v1.5-q8_0.gguf`.

---

## 1) Две функции маршрутизации: `scoreTools` и `selectToolKeys`

Логика маршрутизации намеренно разделена на две маленькие, инспектируемые функции. Вы можете тестировать их независимо и заменять любую, не затрагивая другую.

**`scoreTools`** — запрос vs. все примеры, возвращает оценку для каждого инструмента:

```javascript
function scoreTools(queryEmbedding, exemplarRows) {
    const maxByTool = new Map();
    for (const row of exemplarRows) {
        const sim = queryEmbedding.calculateCosineSimilarity(row.embedding);
        const prev = maxByTool.get(row.toolKey) ?? -Infinity;
        if (sim > prev) maxByTool.set(row.toolKey, sim);
    }
    return maxByTool; // Map<toolKey, maxCosineSimilarity>
}
```

Каждый инструмент может иметь несколько строк-примеров. Используемая оценка — это **максимум** среди всех них, а не среднее. Одно сильное совпадение примера сохраняет инструмент конкурентоспособным, даже если другие формулировки промахиваются — это та же стратегия «max-sim», которая используется в моделях извлечения с поздним взаимодействием.

**`selectToolKeys`** — выбор top-k инструментов плюс закреплённые:

```javascript
function selectToolKeys(scores, k, alwaysInclude) {
    const ranked = [...scores.entries()]
        .sort((a, b) => b[1] - a[1])
        .map(([key]) => key);

    const out = new Set(alwaysInclude);          // start with pinned tools
    for (const key of ranked) {
        if (out.size >= alwaysInclude.size + k) break;
        if (alwaysInclude.has(key)) continue;    // pinned tools don't use a slot
        out.add(key);
    }
    return out;
}
```

Закреплённые инструменты добавляются первыми и **не** потребляют слот извлечения. Запрос `k=1` с одним закреплённым инструментом даёт два инструмента суммарно, а не один.

---

## 2) Каталог инструментов (12 инструментов IT-помощника)

Каждый инструмент — это запись `defineChatSessionFunction` с описанием, JSON-схемой и обработчиком. Обработчики возвращают зафиксированный JSON, чтобы урок оставался сфокусированным на маршрутизации, а не на бизнес-логике.

```javascript
const checkVPNStatus = defineChatSessionFunction({
    description: "Check VPN client status, the active profile, and tunnel health for a user.",
    params: {
        type: "object",
        properties: {
            username: { type: "string", description: "Username to look up (optional)" },
        },
    },
    async handler({ username = "current_user" } = {}) {
        return JSON.stringify({
            username,
            vpn_client: "Cisco AnyConnect 4.10",
            connected: false,
            last_connected: "2026-05-14T09:31:00Z",
            error: "Authentication timeout - check MFA token or try reconnecting",
            profile: "Corp-HQ",
        });
    },
});
```

Двенадцать инструментов покрывают пять различных семантических кластеров. Различные кластеры важны: маршрутизация работает, потому что сходство эмбеддингов разделяет их.

| Кластер | Инструменты |
|---------|-------------|
| Связность | `checkNetworkConnectivity`, `checkVPNStatus` |
| Здоровье системы | `getSystemLogs`, `runDiagnostic`, `getHardwareInfo` |
| Хранилище / софт | `checkDiskSpace`, `getInstalledSoftware` |
| Аккаунт / доступ | `getUserAccount`, `resetPassword` |
| Операции | `restartService`, `createSupportTicket`, `escalateToSpecialist` |

---

## 3) `EXEMPLARS` и холодный старт эмбеддингов

`EXEMPLARS` — плоский список пар `{ toolKey, text }` — три формулировки на инструмент (36 суммарно):

```javascript
const EXEMPLARS = [
    // checkVPNStatus
    { toolKey: "checkVPNStatus", text: "remote tunnel to corporate network not establishing from outside office" },
    { toolKey: "checkVPNStatus", text: "VPN client shows disconnected even after entering valid credentials" },
    { toolKey: "checkVPNStatus", text: "AnyConnect auth keeps timing out when working remotely" },

    // getUserAccount
    { toolKey: "getUserAccount", text: "account suspended or locked in Active Directory after failed logins" },
    { toolKey: "getUserAccount", text: "verify user permissions and group membership in the directory" },
    { toolKey: "getUserAccount", text: "sign-in blocked, need to confirm account standing" },
    // ...
];
```

Строки **не** скопированы из демо-промптов — это парафразы. Если бы примеры были идентичны живым запросам, маршрутизация выглядела бы «магической», но Вы тестировали бы дословный recall, а не семантическое обобщение.

При запуске каждый пример эмбеддинируется один раз и кэшируется:

```javascript
const exemplarRows = [];
for (const { toolKey, text } of EXEMPLARS) {
    const embedding = await embedContext.getEmbeddingFor(text);
    exemplarRows.push({ toolKey, exemplarText: text, embedding });
}
```

В production-сервисе Вы бы персистили эти векторы и пересобирали их только при изменении каталога инструментов.

---

## 4) Настройка модели: один экземпляр `llama`, две модели

Обе модели загружаются через один хэндл `llama`, но каждая получает свой тип контекста:

```javascript
const llama = await getLlama({ debug });

// Chat model - uses a LlamaChatSession
const chatModel = await llama.loadModel({ modelPath: CHAT_MODEL_PATH });
const chatContext = await chatModel.createContext({ contextSize: 4096 });
const session = new LlamaChatSession({
    contextSequence: chatContext.getSequence(),
    chatWrapper: new QwenChatWrapper({ thoughts: "discourage" }),
    systemPrompt: `You are a concise IT helpdesk assistant. ...`,
});

// Embedding model - uses createEmbeddingContext, not a chat context
const embedModel = await llama.loadModel({ modelPath: EMBED_MODEL_PATH });
const embedContext = await embedModel.createEmbeddingContext();
```

`QwenChatWrapper({ thoughts: "discourage" })` предотвращает запись модели Qwen3 пост-инструментального ответа внутри внутренних сегментов «мышления», что привело бы к тому, что `session.prompt()` возвращал бы пустую строку.

---

## 5) `runWithRouting` — маршрутизация + промпт + логирование

Каждый демо-случай вызывает эту функцию. Она запускает полный конвейер и добавляет запись в `routingLog` для визуализации:

```javascript
async function runWithRouting(userPrompt, { retrievalK, alwaysInclude = new Set(), label }) {
    // 1. Embed the live query
    const queryEmbedding = await embedContext.getEmbeddingFor(userPrompt);

    // 2. Score all tools and select the subset
    const scores = scoreTools(queryEmbedding, exemplarRows);
    const selectedKeys = selectToolKeys(scores, retrievalK, alwaysInclude);

    // 3. Print a ranked similarity table (top 6)
    const ranked = [...scores.entries()].sort((a, b) => b[1] - a[1]);
    for (const [key, sim] of ranked.slice(0, 6)) {
        const tick = selectedKeys.has(key) ? "✓" : " ";
        const bar = "█".repeat(Math.round(sim * 24)).padEnd(24);
        console.log(`  [${tick}] ${key.padEnd(28)} ${sim.toFixed(4)}  ${bar}`);
    }

    // 4. Pass only the selected subset to session.prompt
    const functions = pickFunctions(selectedKeys, allFunctions);
    const answer = await session.prompt(userPrompt, { functions, maxTokens: 1200, temperature: 0 });

    // 5. Reset history so each case is independent
    session.resetChatHistory();

    // 6. Store for visualization
    routingLog.push({ label, userPrompt, scores: Object.fromEntries(scores),
                      selectedKeys: [...selectedKeys], alwaysInclude: [...alwaysInclude],
                      retrievalK, answer: answer.trim() });
}
```

Таблица сходства — ключевой учебный вывод. Знак `✓` показывает точно, какие инструменты видит LLM. Все остальные инструменты невидимы для неё на этом ходу.

---

## 6) Демо-случаи (читайте консоль)

Случаи вызываются по порядку. Первые четыре устанавливают базовые показатели; случаи 5 и 6 — **один и тот же промпт** с различными параметрами маршрутизации для демонстрации сбоя recall и того, как закрепление это исправляет:

```javascript
// Cases 1-2: single intent, k=1 - one tool cluster dominates
await runWithRouting(DEMO_VPN,    { retrievalK: 1, label: "Single intent - VPN failure ..." });
await runWithRouting(DEMO_LOCKED, { retrievalK: 1, label: "Single intent - locked account ..." });

// Cases 3-4: dual intent, k=2 - two clusters both needed
await runWithRouting(DEMO_CRASHING_PC,      { retrievalK: 2, label: "Dual intent - crashing PC ..." });
await runWithRouting(DEMO_DISK_AND_SOFTWARE, { retrievalK: 2, label: "Dual intent - disk full + software ..." });

// Case 5: dual intent, k=1 - recall failure: VPN dominates, account tool is cut
await runWithRouting(DEMO_VPN_AND_LOCKED_K1, {
    retrievalK: 1,
    label: "Dual intent - VPN + locked account, k=1: recall failure ...",
});

// Case 6: same prompt, k=1 + pinned getUserAccount - both intents covered without raising k
await runWithRouting(DEMO_VPN_AND_LOCKED_K1, {
    retrievalK: 1,
    alwaysInclude: new Set(["getUserAccount"]),
    label: "Dual intent - same query, k=1 + pinned getUserAccount ...",
});
```

| # | Намерение | k | Закреплённые | Ожидаемые видимые инструменты |
|---|-----------|---|--------------|-------------------------------|
| 1 | VPN-сбой | 1 | - | `checkVPNStatus` |
| 2 | Заблокированный аккаунт | 1 | - | `getUserAccount` или `resetPassword` |
| 3 | Падение ПК | 2 | - | `getSystemLogs` + `runDiagnostic` |
| 4 | Диск полон + софт | 2 | - | `checkDiskSpace` + `getInstalledSoftware` |
| 5 | VPN + заблокированный аккаунт | 1 | - | Только VPN — **инструмент аккаунта отсутствует** |
| 6 | VPN + заблокированный аккаунт | 1 | `getUserAccount` | Оба покрыты |

---

## 7) Очистка

Ресурсы освобождаются в обратном порядке зависимостей:

```javascript
session.dispose();
chatContext.dispose();
chatModel.dispose();
embedContext.dispose();
embedModel.dispose();
llama.dispose();
```
