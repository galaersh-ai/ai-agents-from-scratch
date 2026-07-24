# Объяснение кода: `chain-of-thought.js`

Это руководство отображает каждую фазу CoT на фактические функции в файле.

## Запуск

```bash
node examples/14_chain-of-thought/chain-of-thought.js
```

---

## 1) Настройка: модель, входной случай и схемы

В верхнем уровне файла:

- `RETURN_CASE` определяет запрос клиента.
- `RETURN_POLICY` определяет жёсткие бизнес-ограничения.
- `factsSchema`, `redFlagsSchema`, `legitimacySchema`, `policySchema`, `decisionSchema` определяют JSON-контракт для каждой фазы.
- `promptJson(schema, userText)` — общая утилита, которая:
  - сбрасывает историю чата,
  - обеспечивает грамматику схемы,
  - безопасно парсит и исправляет JSON.

Это даёт каждой функции фазы строгую форму вывода.

```js
const RETURN_CASE = {
    request_id: "RET-2026-0414",
    claimed_reason: "Right ear cup has intermittent sound dropouts",
    claim_timing_days_after_delivery: 23,
    order_value_eur: 189.0
    // ...
};

const RETURN_POLICY = {
    return_window_days: 30,
    max_high_value_returns_12m_before_manual_review: 2,
    mandatory_manual_review_amount_eur: 250
};

async function promptJson(schema, userText) {
    session.resetChatHistory();
    const grammar = await llama.createGrammarForJsonSchema(schema);
    const raw = await session.prompt(userText, { grammar, maxTokens: 1400, temperature: 0.2 });
    return JsonParser.parse(raw, { debug, expectObject: true, repairAttempts: true });
}
```

---

## 2) Фаза 1 (Факты): `extractFacts()`

`extractFacts(returnCase)` запрашивает:

- только явные факты,
- без оценок,
- без суждений.

Возвращает:

- `extracted_facts`
- `missing_information`

Это защищает от раннего предвзятости до начала рассуждений о рисках.

```js
async function extractFacts(returnCase) {
    return promptJson(
        factsSchema,
        `Phase 1 of 5: FACTS ONLY.
Extract facts from the return request without evaluation, suspicion, or judgment.
Do not infer intent. Do not score. Just capture what is explicitly known.

Return request JSON:
${JSON.stringify(returnCase, null, 2)}`
    );
}
```

---

## 3) Фаза 2 (Тревожные сигналы): `screenRedFlags()`

`screenRedFlags(returnCase, facts)` выполняет явный скрининг мошенничества с фиксированными чекпоинтами.

Вывод:

- `checkpoints[]` с `present/not_present/unclear`
- `fraud_score`
- `fraud_rationale`

Важная часть — покрытие чек-листа, а не просто одна финальная оценка.

```js
async function screenRedFlags(returnCase, facts) {
    return promptJson(
        redFlagsSchema,
        `Phase 2 of 5: RED FLAG SCREENING.
Evaluate potential fraud indicators one by one.

Use these checkpoints:
1) Frequent recent return behavior
2) High-value return pattern
3) Inconsistent payment/shipping identity
4) Weak or missing defect evidence
5) Timing pattern that looks strategic
6) Account behavior anomaly`
    );
}
```

---

## 4) Фаза 3 (Легитимность): `assessLegitimacy()`

`assessLegitimacy(returnCase, facts)` строит аргумент со стороны клиента:

- правдоподобные индикаторы дефекта,
- факторы справедливости/контекста,
- качество подтверждающих доказательств.

Вывод:

- `customer_supporting_points[]`
- `legitimacy_score`
- `legitimacy_rationale`

Без этой фазы логика рисков tends доминирует в каждом пограничном случае.

```js
async function assessLegitimacy(returnCase, facts) {
    return promptJson(
        legitimacySchema,
        `Phase 3 of 5: LEGITIMACY VIEW.
Now build the customer-side case.
List reasons why this may be a legitimate return.
Do not reference fraud score. Focus on fairness and plausible product failure.`
    );
}
```

---

## 5) Фаза 4 (Политика): `checkPolicy()`

`checkPolicy(returnCase, policy, redFlags, legitimacy)` применяет жёсткие правила:

- окно возврата
- пороги стоимости
- триггеры по истории возвратов

Вывод:

- статусы по правилам в `policy_checks[]`
- `policy_outcome` (`approve`, `reject`, `manual_review`)

Это ворота управления между анализом и действием.

```js
async function checkPolicy(returnCase, policy, redFlags, legitimacy) {
    return promptJson(
        policySchema,
        `Phase 4 of 5: POLICY CHECK.
Apply policy strictly. Do not invent rules.

Policy JSON:
${JSON.stringify(policy, null, 2)}

Fraud score: ${redFlags.fraud_score}
Legitimacy score: ${legitimacy.legitimacy_score}`
    );
}
```

---

## 6) Фаза 5 (Решение): `makeDecision()`

`makeDecision(...)` может решить только после всех предыдущих фаз.

Вывод:

- `final_decision`
- `confidence`
- `decision_reasoning`
- `customer_message`
- `internal_note`

Промпт явно ссылается на обработку конфликтов (например, мошенничество 6/10 vs легитимность 7/10), поэтому результат должен объяснять, как политика разрешает напряжение.

```js
async function makeDecision(returnCase, phase1Facts, redFlags, legitimacy, policyResult) {
    return promptJson(
        decisionSchema,
        `Phase 5 of 5: FINAL DECISION.
You can decide only now. Use all prior phases.
Explain trade-offs clearly. If conflict exists (e.g., fraud 6/10 vs legitimacy 7/10),
show how policy resolves it.`
    );
}
```

---

## 7) Поток оркестрации: `runChainOfThoughtReturnDecision()`

Главный контроллер выполняет фазы в строгом порядке:

1. факты
2. тревожные сигналы
3. легитимность
4. проверка политик
5. финальное решение

Затем печатает компактный отчёт и записывает визуализацию для браузера через:

- `writeCoTReturnVisualization(...)`

Это позволяет основному файлу оставаться сфокусированным на логике CoT.

```js
async function runChainOfThoughtReturnDecision(returnCase, policy) {
    const facts = await extractFacts(returnCase);
    const redFlags = await screenRedFlags(returnCase, facts);
    const legitimacy = await assessLegitimacy(returnCase, facts);
    const policyResult = await checkPolicy(returnCase, policy, redFlags, legitimacy);
    const decision = await makeDecision(returnCase, facts, redFlags, legitimacy, policyResult);

    writeCoTReturnVisualization(__dirname, {
        returnCase, policy, facts, redFlags, legitimacy, policyResult, decision
    });
}
```

---

## 8) Адаптация реализации под класс модели

Текущий код использует `Qwen3-1.7B-Q8_0.gguf`, который может работать как рассуждающая, так и нерассуждающая модель. Опора из 5 фаз спроектирована для работы с любым классом — но способ настройки различается.

Для концептуальной стороны этого различия см. раздел «CoT с рассуждающими vs нерассуждающими LLM» в [CONCEPT.md](CONCEPT.md).

### Что предполагает текущий код

- Гибридная модель, которая может или не может рассуждать внутренне.
- JSON-схемы для каждой фазы через `promptJson(...)`.
- Низкая `temperature` (0.2) и щедрый бюджет `maxTokens` на каждую фазу.
- Изолированная история чата для каждой фазы через `session.resetChatHistory()`.

Это намеренно средняя конфигурация, чтобы пример работал без необходимости скачивать конкретную модель.

### Настройка для нерассуждающих моделей

Если Вы подставляете базовую/чат-модель без внутренних рассуждений (Llama-3 chat, Phi, Mistral-instruct, Qwen3 с `thoughts: "discourage"`):

```js
const raw = await session.prompt(userText, {
    grammar,
    maxTokens: 1800,
    temperature: 0.1
});
```

- Понизьте `temperature` ещё (0.05 - 0.15). Пограничные случаи сильно деградируют при креативном сэмплировании.
- Увеличьте `maxTokens` на каждую фазу. Модели часто нужно пространство «поговорить с собой» внутри JSON перед тем, как она зафиксирует оценки.
- Держите схемы строгими. Избегайте широких полей свободного формата; заменяйте их перечислениями, фиксированными массивами или короткими ограниченными строками.
- Добавьте явные примеры в промпты фаз («Пример чекпоинта: { check, status, evidence }»). Нерассуждающие модели гораздо быстрее цепляются за форматные примеры, чем за абстрактные спецификации.

### Настройка для рассуждающих моделей

Если Вы подставляете рассуждающую модель (o3, DeepSeek-R1, Qwen3 с `thoughts: "auto"`, Claude Extended Thinking через API):

```js
const raw = await session.prompt(userText, {
    grammar,
    maxTokens: 900,
    temperature: 0.3
});
```

- Сократите промпты фаз. Модель уже рассуждает внутри; многословные инструкции добавляют шум.
- Понизьте `maxTokens` для чисто структурных фаз (Факты, Проверка политик). Им не нужны большие бюджеты на размышление.
- Сохраняйте схемы как **контракт**, а не как подпорку для рассуждений. Их основная задача здесь — межуровневая совместимость.
- Если runtime поддерживает, логируйте внутренний след рассуждений только для отладки — никогда как часть аудит-следа.

### Специфика Qwen3

Для `node-llama-cpp` чистый переключатель поведения мышления Qwen — опция обёртки:

```js
import { QwenChatWrapper } from "node-llama-cpp";

const reasoningWrapper = new QwenChatWrapper({
    thoughts: "auto",
    keepOnlyLastThought: true
});

const nonReasoningWrapper = new QwenChatWrapper({
    thoughts: "discourage"
});
```

Затем создайте чат-сессию с обёрткой, которую Вы хотите для этой фазы/запуска:

```js
const session = new LlamaChatSession({
    contextSequence: context.getSequence(),
    systemPrompt,
    chatWrapper: reasoningWrapper // or nonReasoningWrapper
});
```

Полезный паттерн — смешивать режимы обёрток для разных фаз:

- `thoughts: "discourage"` на фазе 1 (Факты) и фазе 4 (Проверка политик) — механическая работа.
- `thoughts: "auto"` на фазе 2 (Тревожные сигналы), фазе 3 (Легитимность) и фазе 5 (Решение) — работа суждения.

Это сохраняет общую латентность низкой, обеспечивая рассуждения там, где это важно.

### Выноски по фазам

- **Фаза 1 (Факты)** — нерассуждающие модели часто галлюцинируют записи фактов, которые выглядят правдоподобно, но никогда не были во входных данных. Ужесточите схему (`minItems`, поля-перечисления) и инструктируйте явно: «Не предполагайте.»
- **Фаза 2 (Тревожные сигналы)** — рассуждающие модели tend к чрезмерному подозрению при подаче фрейма мошенничества. Закрепите их фиксированным списком чекпоинтов вместо генерации тревожных сигналов с открытым окончанием.
- **Фаза 3 (Легитимность)** — эта фаза существует именно для противодействия предвзятости фазы 2. Не сворачивайте её в фазу 2 для экономии токенов, независимо от класса модели. Это структурный противовес.
- **Фаза 4 (Проверка политик)** — оба класса выигрывают от инъекции политики как inline JSON, а не описания её прозой. Снижает дрейф и молчаливое изобретение правил.
- **Фаза 5 (Решение)** — калибровка уверенности резко различается между классами. `confidence: 0.79` от рассуждающей модели не напрямую сравнимо с `0.79` от базовой модели. Рассматривайте confidence как внутреннее для модели; маршрутизируйте по `final_decision` и `policy_outcome`.

---

## Рекомендуемый порядок чтения кода

1. `promptJson`
2. `extractFacts`
3. `screenRedFlags`
4. `assessLegitimacy`
5. `checkPolicy`
6. `makeDecision`
7. `runChainOfThoughtReturnDecision`

Эта последовательность отражает runtime и делает пример понятным для рассуждений.
