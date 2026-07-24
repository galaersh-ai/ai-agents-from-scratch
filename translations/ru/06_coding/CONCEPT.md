# Концепция: Стриминг и контроль ответов

## Обзор

Этот пример демонстрирует **стриминг ответов** и **лимиты токенов** — две необходимые техники для создания отзывчивых AI-агентов с контролируемым выводом.

## Проблема стриминга

### Традиционный (не-стриминговый) подход

```
User sends prompt
       ↓
   [Wait 10 seconds...]
       ↓
Complete response appears all at once
```

**Проблемы:**
- Плохой пользовательский опыт (длинное ожидание)
- Нет индикации прогресса
- Невозможно прервать плохой ответ
- Выглядит неотзывчиво

### Стриминговый подход (этот пример)

```
User sends prompt
       ↓
"Hoisting" (0.1s) → User sees first word!
       ↓
"is a" (0.2s) → More text appears
       ↓
"JavaScript" (0.3s) → Continuous feedback
       ↓
[Continues token by token...]
```

**Преимущества:**
- Немедленная обратная связь
- Видимый прогресс
- Можно прервать рано
- Выглядит интерактивно

## Как работает стриминг

### Поколенная генерация токенов

LLM генерируют по одному токену за раз. Стриминг это обнажает:

```
Internal LLM Process:
┌─────────────────────────────────────┐
│  Token 1: "Hoisting"                │
│  Token 2: "is"                      │
│  Token 3: "a"                       │
│  Token 4: "JavaScript"              │
│  Token 5: "mechanism"               │
│  ...                                │
└─────────────────────────────────────┘

Without Streaming:        With Streaming:
Wait for all tokens       Emit each token immediately
└─→ Buffer → Return      └─→ Callback → Display
```

### Колбэк onTextChunk

```
┌────────────────────────────────────┐
│        Model Generation            │
└────────────┬───────────────────────┘
             │
    ┌────────┴─────────┐
    │  Each new token  │
    └────────┬─────────┘
             ↓
    ┌────────────────────┐
    │ onTextChunk(text)  │  ← Your callback
    └────────┬───────────┘
             ↓
    Your code processes it:
    • Display to user
    • Send over network
    • Log to file
    • Analyze content
```

## Лимиты токенов: maxTokens

### Зачем ограничивать вывод?

Без лимитов модели могут генерировать:
```
User: "Explain hoisting"
Model: [Generates 10,000 words including:
        - Complete JavaScript history
        - Every edge case
        - Unrelated examples
        - Never stops...]
```

С лимитами:
```
User: "Explain hoisting"
Model: [Generates ~1500 words
        - Core concept
        - Key examples
        - Stops at 2000 tokens]
```

### Бюджет токенов

```
Context Window: 4096 tokens
├─ System Prompt: 200 tokens
├─ User Message: 100 tokens
├─ Response (maxTokens): 2000 tokens
└─ Remaining for history: 1796 tokens

Total used: 2300 tokens
Available: 1796 tokens for future conversation
```

### Стоимость vs. Качество

```
Token Limit        Output Quality      Use Case
───────────       ───────────────     ─────────────────
100               Brief, may be cut   Quick answers
500               Concise but complete Short explanations
2000 (example)    Detailed            Full explanations
No limit          Risk of rambling    When length unknown
```

## Приложения в реальном времени

### Паттерн 1: Интерактивный CLI

```
User: "Explain closures"
       ↓
Terminal: "A closure is a function..."
         (Appears word by word, like typing)
       ↓
User sees progress, knows it's working
```

### Паттерн 2: Веб-приложение

```
Browser                    Server
   │                         │
   ├─── Send prompt ────────→│
   │                         │
   │←── Chunk 1: "Closures"──┤
   │    (Display immediately) │
   │                         │
   │←── Chunk 2: "are"───────┤
   │    (Append to display)  │
   │                         │
   │←── Chunk 3: "functions"─┤
   │    (Keep appending...)  │
```

Реализация:
- Server-Sent Events (SSE)
- WebSockets
- HTTP стриминг

### Паттерн 3: Мульти-потребитель

```
         onTextChunk(text)
                │
        ┌───────┼───────┐
        ↓       ↓       ↓
    Console  WebSocket  Log File
    Display  → Client   → Storage
```

## Характеристики производительности

### Латентность vs. Пропускная способность

```
Time to First Token (TTFT):
├─ Small model (1.7B): ~100ms
├─ Medium model (8B): ~200ms
└─ Large model (20B): ~500ms

Tokens Per Second:
├─ Small model: 50-80 tok/s
├─ Medium model: 20-35 tok/s
└─ Large model: 10-15 tok/s

User Experience:
TTFT < 500ms → Feels instant
Tok/s > 20 → Reads naturally
```

### Компромиссы ресурсов

```
Model Size      Memory    Speed     Quality
──────────     ────────   ─────     ───────
1.7B           ~2GB       Fast      Good
8B             ~6GB       Medium    Better
20B            ~12GB      Slower    Best
```

## Продвинутые концепции

### Стратегии буферизации

**Без буфера (немедленно)**
```
Every token → callback → display
└─ Smoothest UX but more overhead
```

**Строковый буфер**
```
Accumulate until newline → flush
└─ Better for paragraph-based output
```

**Временной буфер**
```
Accumulate for 50ms → flush batch
└─ Reduces callback frequency
```

### Ранняя остановка

```
Generation in progress:
"The answer is clearly... wait, actually..."
                         ↑
                  onTextChunk detects issue
                         ↓
                   Stop generation
                         ↓
              "Let me reconsider"
```

Полезно для:
- Обнаружения ответов не по теме
- Фильтров безопасности
- Проверки релевантности

### Прогрессивное улучшение

```
Partial Response Analysis:
┌─────────────────────────────────┐
│ "To implement this feature..."  │
│                                 │
│ ← Already useful information   │
│                                 │
│ "...you'll need: 1) Node.js"    │
│                                 │
│ ← Can start acting on this     │
│                                 │
│ "2) Express framework"          │
└─────────────────────────────────┘

Agent can begin working before response completes!
```

## Осведомлённость о размере контекста

### Почему это важно

```
┌────────────────────────────────┐
│    Context Window (4096)       │
├────────────────────────────────┤
│ System Prompt       200 tokens │
│ Conversation History 1000      │
│ Current Prompt      100        │
│ Response Space      2796       │
└────────────────────────────────┘

If maxTokens > 2796:
└─→ Error or truncation!
```

### Динамическая настройка

```
Available = contextSize - (prompt + history)

if (maxTokens > available) {
    maxTokens = available;
    // or clear old history
}
```

## Стриминг в архитектурах агентов

### Простой агент

```
User → LLM (streaming) → Display
       └─ onTextChunk shows progress
```

### Многошаговый агент

```
Step 1: Plan (stream) → Show thinking
Step 2: Act (stream) → Show action
Step 3: Result (stream) → Show outcome
       └─ User sees agent's process
```

### Совместные агенты

```
Agent A (streaming) ──┐
                      ├─→ Coordinator → User
Agent B (streaming) ──┘
       └─ Both stream simultaneously
```

## Лучшие практики

### 1. Всегда устанавливайте maxTokens

```
✓ Good:
session.prompt(query, { maxTokens: 2000 })

✗ Risky:
session.prompt(query)
└─ May use entire context!
```

### 2. Обрабатывайте частичные обновления

```
let fullResponse = '';
onTextChunk: (chunk) => {
    fullResponse += chunk;
    display(chunk);        // Show immediately
    logComplete = false;   // Mark incomplete
}
// After completion:
saveToDatabase(fullResponse);
```

### 3. Предоставляйте обратную связь

```
onTextChunk: (chunk) => {
    if (firstChunk) {
        showLoadingDone();
        firstChunk = false;
    }
    appendToDisplay(chunk);
}
```

### 4. Мониторьте производительность

```
const startTime = Date.now();
let tokenCount = 0;

onTextChunk: (chunk) => {
    tokenCount += estimateTokens(chunk);
    const elapsed = (Date.now() - startTime) / 1000;
    const tokensPerSecond = tokenCount / elapsed;
    updateMetrics(tokensPerSecond);
}
```

## Ключевые выводы

1. **Стриминг улучшает UX**: Пользователи видят прогресс немедленно
2. **maxTokens контролирует стоимость**: Предотвращает бесконечную генерацию
3. **Поколенная генерация**: LLM производят по одному токену
4. **Колбэк onTextChunk**: Ваш хук в процесс генерации
5. **Осведомлённость о контексте важна**: Мониторьте доступное пространство
6. **Необходимо для продакшена**: Системы реального времени нуждаются в стриминге

## Сравнение

```
Feature           intro.js    coding.js (this)
────────────────  ─────────   ─────────────────
Streaming         ✗           ✓
Token limit       ✗           ✓ (2000)
Real-time output  ✗           ✓
Progress visible  ✗           ✓
User control      ✗           ✓
```
