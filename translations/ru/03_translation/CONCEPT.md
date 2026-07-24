# Концепция: Системные промпты и специализация агентов

## Обзор

Этот пример демонстрирует, как превратить универсальную LLM в **специализированного агента** с помощью **системных промптов**. Ключевой вывод: Вам не нужны разные модели для разных задач — нужны разные инструкции.

## Что такое системный промпт?

**Системный промпт** — это постоянная инструкция, которая формирует поведение AI на протяжении всей сессии разговора.

### Аналогия

Представьте найм сотрудника на работу:

```
Without System Prompt          With System Prompt
─────────────────────         ──────────────────────
"Hi, I'm an AI."              "I'm a professional translator
                               with expertise in scientific
"What can I do for you?"            German. I follow strict quality
                              guidelines and output format."
```

## Как работают системные промпты

### Структура контекста

```
┌─────────────────────────────────────────────┐
│           CONTEXT WINDOW                    │
│                                             │
│  ┌───────────────────────────────────────┐ │
│  │  SYSTEM PROMPT (Always present)       │ │
│  │  "You are a professional translator..." │
│  │  "Follow these rules..."              │ │
│  └───────────────────────────────────────┘ │
│                    ↓                        │
│  ┌───────────────────────────────────────┐ │
│  │  USER MESSAGES                        │ │
│  │  "Translate this text..."             │ │
│  └───────────────────────────────────────┘ │
│                    ↓                        │
│  ┌───────────────────────────────────────┐ │
│  │  AI RESPONSES                         │ │
│  │  (Shaped by system prompt)            │ │
│  └───────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

Системный промпт находится в начале контекста и влияет на **каждый** ответ.

## Паттерн специализации агентов

### Поток трансформации

```
┌──────────────────┐    ┌─────────────────┐    ┌──────────────────┐
│  General Model   │ +  │ System Prompt   │ =  │ Specialized Agent│
│                  │    │                 │    │                  │
│ • Knows many     │    │ • Define role   │    │ • Translation    │
│   things         │    │ • Set rules     │    │   Agent          │
│ • No specific    │    │ • Constrain     │    │ • Coding Agent   │
│   role           │    │   output        │    │ • Analysis Agent │
└──────────────────┘    └─────────────────┘    └──────────────────┘
```

### Примеры специализаций

**Агент-переводчик (этот пример):**
```
System Prompt = Role + Rules + Output Format
```

**Кодирующий ассистент:**
```javascript
systemPrompt: "You are an expert programmer. 
Always provide working code with comments.
Explain complex logic."
```

**Аналитик данных:**
```javascript
systemPrompt: "You are a data analyst.
Always show your calculations step-by-step.
Cite data sources when available."
```

## Структура эффективного системного промпта

### 5 компонентов

```
┌─────────────────────────────────────────┐
│  1. ROLE DEFINITION                     │
│  "You are a [specific role]..."         │
├─────────────────────────────────────────┤
│  2. TASK DESCRIPTION                    │
│  "Your goal is to..."                   │
├─────────────────────────────────────────┤
│  3. BEHAVIORAL RULES                    │
│  "Always do X, Never do Y..."           │
├─────────────────────────────────────────┤
│  4. OUTPUT FORMAT                       │
│  "Format your response as..."           │
├─────────────────────────────────────────┤
│  5. CONSTRAINTS                         │
│  "Do NOT include..."                    │
└─────────────────────────────────────────┘
```

### Структура этого примера

```
Role:        "Professional scientific translator"
Task:        "Translate English to German with precision"
Rules:       8 specific translation guidelines
Format:      Idiomatic German, scientific style
Constraints: "ONLY translated text, no explanation"
```

## Почему детальные системные промпты важны

### Сравнительное исследование

**Минимальный системный промпт:**
```javascript
systemPrompt: "Translate to German"
```

**Результат:**
- Может добавлять ненужные объяснения
- Несогласованная терминология
- Смешанные уровни формальности
- Лишний разговорный текст

**Детальный системный промпт (этот пример):**
```javascript
systemPrompt: `You are a professional translator...
- Rule 1: Preserve technical accuracy
- Rule 2: Use idiomatic German
- Rule 3: Follow scientific conventions
...
DO NOT add any explanations`
```

**Результат:**
- ✅ Согласованное качество
- ✅ Правильная терминология
- ✅ Корректное форматирование
- ✅ Только вывод перевода

### Влияние на качество

```
Detail Level          Output Quality
───────────         ─────────────────
Very minimal  →     Unpredictable
Basic role    →     Somewhat consistent
Detailed      →     Highly consistent ⭐
Over-detailed →     May confuse model
```

## Паттерны проектирования системных промптов

### Паттерн 1: Ролевая игра
```
"You are a [profession] with expertise in [domain]..."
```
Заставляет модель adoptировать эту перспективу.

### Паттерн 2: На основе правил
```
"Follow these rules:
1. Always...
2. Never...
3. When X, do Y..."
```
Явные ограничения ведут к предсказуемому поведению.

### Паттерн 3: Форматирование вывода
```
"Format your response as:
- JSON
- Markdown
- Plain text only
- Step-by-step list"
```
Контролирует структуру ответов.

### Паттерн 4: Контекстная осведомлённость
```
"You remember: [previous facts]
You know that: [domain knowledge]
Current situation: [context]"
```
Подготавливает модель соответствующей информацией.

## Как это связано с AI-агентами

### Агент = Модель + Системный промпт + Инструменты

```
┌────────────────────────────────────────────┐
│             AI Agent                       │
│                                            │
│  ┌──────────────────────────────────────┐ │
│  │  System Prompt (Agent's "Identity")  │ │
│  └──────────────────────────────────────┘ │
│                  ↓                         │
│  ┌──────────────────────────────────────┐ │
│  │  LLM (Agent's "Brain")               │ │
│  └──────────────────────────────────────┘ │
│                  ↓                         │
│  ┌──────────────────────────────────────┐ │
│  │  Tools (Agent's "Hands") [Optional]  │ │
│  └──────────────────────────────────────┘ │
└────────────────────────────────────────────┘
```

**В этом примере:**
- Системный промпт: «Вы переводчик...»
- LLM: Модель Apertus-8B
- Инструменты: Нет (перевод выполняется самой моделью)

**В более сложных агентах:**
- Системный промпт: «Вы исследовательский ассистент...»
- LLM: Любая модель
- Инструменты: Веб-поиск, калькулятор, доступ к файлам и т.д.

## Практические применения

### 1. Специализация по области
```
Medical → "You are a medical professional..."
Legal → "You are a legal expert..."
Technical → "You are an engineer..."
```

### 2. Контроль вывода
```
JSON API → "Always respond in valid JSON"
Markdown → "Format all responses as markdown"
Code → "Only output executable code"
```

### 3. Поведенческие ограничения
```
Concise → "Use maximum 2 sentences"
Detailed → "Explain thoroughly with examples"
Neutral → "Avoid opinions, state only facts"
```

### 4. Многоязычная поддержка
```
systemPrompt: `You are a multilingual assistant.
Respond in the same language as the input.`
```

## Chat-обёртки

Разным моделям нужны разные форматы разговора:

```
Model Type        Format Needed         Wrapper
──────────────   ───────────────────   ─────────────────
Llama 2/3        Llama format          LlamaChatWrapper
GPT-style        ChatML format         ChatMLWrapper
Harmony models   Harmony format        HarmonyChatWrapper
```

**Что они делают:**
```
Your Message → [Chat Wrapper] → Formatted Prompt → Model
                    ↓
          Adds special tokens:
          <|system|>, <|user|>, <|assistant|>
```

Обёртка обеспечивает понимание моделью, какая часть является системным промптом, какая — сообщением пользователя и т.д.

## Ключевые выводы

1. **Системные промпты мощные**: они фундаментально меняют поведение модели
2. **Детальность лучше**: более конкретные инструкции = более согласованные результаты
3. **Структура важна**: Роль + Правила + Формат + Ограничения
4. **Не нужно дообучение**: одна модель, разные поведения
5. **Фундамент для агентов**: системные промпты — первый шаг в создании специализированных агентов

## Путь эволюции

```
1. Basic Prompting           (intro.js)
       ↓
2. System Prompts            (translation.js) ← You are here
       ↓
3. System Prompts + Tools    (simple-agent.js)
       ↓
4. Multi-turn reasoning      (react-agent.js)
       ↓
5. Full Agent Systems
```

Этот пример преодолевает разрыв между базовым использованием LLM и истинным поведением агента, показывая, как специализировать через инструкции.
