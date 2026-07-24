# Концепция: Рассуждающие агенты и решение задач

## Обзор

Этот пример демонстрирует, как настроить LLM как **рассуждающего агента**, способного к аналитическому мышлению и решению количественных задач. Он показывает мост между простой генерацией текста и сложными когнитивными задачами.

## Что такое рассуждающий агент?

**Рассуждающий агент** — это LLM, настроенная для выполнения логического анализа, математических вычислений и многошагового решения задач через тщательное проектирование системного промпта.

### Человеческая аналогия

```
Regular Chat                    Reasoning Agent
─────────────                  ──────────────────
"Can you help me?"            "I am a mathematician.
"Sure! What do you need?"     I analyze problems methodically
                              and compute exact answers."
```

## Проблема рассуждений

### Почему рассуждения сложны для LLM

LLM обучены на предсказании текста, а не на явных рассуждениях:

```
┌───────────────────────────────────────┐
│  LLM Training                         │
│  "Predict next word in text"         │
│                                       │
│  NOT explicitly trained for:         │
│  • Step-by-step logic                │
│  • Arithmetic computation            │
│  • Tracking multiple variables       │
│  • Systematic problem decomposition  │
└───────────────────────────────────────┘
```

Тем не менее, они могут выучить паттерны рассуждений из обучающих данных и быть направлены системными промптами.

## Рассуждения через системные промпты

### Паттерн конфигурации

```
┌─────────────────────────────────────────┐
│  System Prompt Components              │
├─────────────────────────────────────────┤
│  1. Role: "Expert reasoner"            │
│  2. Task: "Analyze and solve problems" │
│  3. Method: "Compute exact answers"    │
│  4. Output: "Single numeric value"     │
└─────────────────────────────────────────┘
         ↓
   Reasoning Behavior
```

### Типы задач рассуждений

**Количественные рассуждения (этот пример):**
```
Problem → Count entities → Calculate → Convert units → Answer
```

**Логические рассуждения:**
```
Premises → Apply rules → Deduce conclusions → Answer
```

**Аналитические рассуждения:**
```
Data → Identify patterns → Form hypothesis → Conclude
```

## Как LLM «рассуждают»

### Сопоставление паттернов vs. Истинные рассуждения

LLM не рассуждают как люди, но они могут:

```
┌─────────────────────────────────────────────┐
│  What LLMs Actually Do                      │
│                                             │
│  1. Pattern Recognition                     │
│     "This looks like a counting problem"    │
│                                             │
│  2. Template Application                    │
│     "Similar problems follow this pattern"  │
│                                             │
│  3. Statistical Inference                   │
│     "These numbers likely combine this way" │
│                                             │
│  4. Learned Procedures                      │
│     "I've seen this type of calculation"    │
└─────────────────────────────────────────────┘
```

### Процесс рассуждений

```
Input: Complex Word Problem
         ↓
    ┌────────────┐
    │   Parse    │  Identify entities and relationships
    └────────────┘
         ↓
    ┌────────────┐
    │  Decompose │  Break into sub-problems
    └────────────┘
         ↓
    ┌────────────┐
    │  Calculate │  Apply arithmetic operations
    └────────────┘
         ↓
    ┌────────────┐
    │  Synthesize│  Combine results
    └────────────┘
         ↓
     Final Answer
```

## Иерархия сложности задач

### Уровни сложности рассуждений

```
Easy                                        Hard
│                                             │
│  Simple    Multi-step   Nested    Implicit │
│  Arithmetic  Logic    Conditions  Reasoning│
│                                             │
└─────────────────────────────────────────────┘

Examples:
Easy:    "What is 5 + 3?"
Medium:  "If 3 apples cost $2 each, what's the total?"
Hard:    "Count family members with complex relationships"
```

### Сложность этого примера

Задача с картофелем **очень сложная**:

```
┌─────────────────────────────────────────┐
│  Complexity Factors                     │
├─────────────────────────────────────────┤
│  ✓ Multiple entities (15+ people)      │
│  ✓ Relationship reasoning (family tree)│
│  ✓ Conditional logic (if married then..)│
│  ✓ Negative conditions (deceased people)│
│  ✓ Special cases (dietary restrictions)│
│  ✓ Multiple calculations                │
│  ✓ Unit conversions                     │
└─────────────────────────────────────────┘
```

## Ограничения чистых LLM-рассуждений

### Почему этот подход имеет проблемы

```
┌────────────────────────────────────┐
│  Problem: No External Tools        │
│                                    │
│  LLM must hold everything in       │
│  "mental" context:                 │
│  • All entity counts               │
│  • Intermediate calculations       │
│  • Conversion factors              │
│  • Final arithmetic                │
│                                    │
│  Result: Prone to errors           │
└────────────────────────────────────┘
```

### Типичные режимы сбоев

**1. Ошибки подсчёта:**
```
Problem: "Count 15 people with complex relationships"
LLM: "14" or "16" (off by one)
```

**2. Арифметические ошибки:**
```
Problem: "13 adults × 1.5 + 3 kids × 0.5"
LLM: May get intermediate steps wrong
```

**3. Потеря контекста:**
```
Problem: Multi-step with many facts
LLM: Forgets earlier information
```

## Улучшение рассуждений: Путь эволюции

### Уровень 1: Чистый промптинг (этот пример)
```
User → LLM → Answer
       ↑
   System Prompt
```

**Ограничения:**
- Все рассуждения внутри LLM
- Нет верификации
- Нет инструментов
- Скрытый процесс

### Уровень 2: Цепочка рассуждений
```
User → LLM → Show Work → Answer
       ↑
   "Explain your reasoning"
```

**Улучшения:**
- Видимые шаги рассуждений
- Можно поймать некоторые ошибки
- Всё ещё нет инструментов

### Уровень 3: С инструментами (simple-agent)
```
User → LLM ⟷ Tools → Answer
       ↑    (Calculator)
   System Prompt
```

**Улучшения:**
- Внешние вычисления
- Меньше ошибок
- Верифицируемые шаги

### Уровень 4: Паттерн ReAct (react-agent)
```
User → LLM → Think → Act → Observe
       ↑      ↓      ↓      ↓
   System  Reason  Tool   Result
   Prompt         Use
       ↑           ↓       ↓
       └───────────Iterate──┘
```

**Лучший подход:**
- Явный цикл рассуждений
- Использование инструментов на каждом шаге
- Возможна самокоррекция

## Проектирование системного промпта для рассуждений

### Ключевые элементы

**1. Определение роли:**
```
"You are an expert logical and quantitative reasoner"
```
Устанавливает ментальную рамку.

**2. Спецификация задачи:**
```
"Analyze real-world word problems involving..."
```
Определяет область задач.

**3. Формат вывода:**
```
"Return the correct final number as a single value"
```
Контролирует структуру ответа.

### Паттерны проектирования

**Паттерн A: Прямой ответ (этот пример)**
```
Prompt: [Problem]
Output: [Number]
```
Плюсы: Краткий, быстрый
Минусы: Нет понимания рассуждений

**Паттерн B: Показать работу**
```
Prompt: [Problem] "Show your steps"
Output: Step 1: ... Step 2: ... Answer: [Number]
```
Плюсы: Прозрачный, отлаживаемый
Минусы: Длиннее, всё ещё могут быть ошибки

**Паттерн C: Самоверификация**
```
Prompt: [Problem] "Solve, then verify"
Output: Solution + Verification + Final Answer
```
Плюсы: Надёжнее
Минусы: Медленнее, больше токенов

## Практические применения

### Случаи использования рассуждающих агентов

**1. Анализ данных:**
```
Input: Dataset summary
Task: Compute statistics, identify trends
Output: Numerical insights
```

**2. Планирование:**
```
Input: Goal + constraints
Task: Reason about optimal sequence
Output: Action plan
```

**3. Поддержка принятия решений:**
```
Input: Options + criteria
Task: Evaluate and compare
Output: Recommended choice
```

**4. Решение задач:**
```
Input: Complex scenario
Task: Break down and solve
Output: Solution
```

## Сравнение: Разные типы агентов

```
                  Reasoning  Tools  Memory  Multi-turn
                  ─────────  ─────  ──────  ──────────
intro.js              ✗        ✗      ✗        ✗
translation.js        ~        ✗      ✗        ✗
think.js (here)       ✓        ✗      ✗        ✗
simple-agent.js       ✓        ✓      ✗        ~
memory-agent.js       ✓        ✓      ✓        ✓
react-agent.js        ✓✓       ✓      ~        ✓
```

Легенда:
- ✗ = Отсутствует
- ~ = Ограниченное/неявное
- ✓ = Присутствует
- ✓✓ = Продвинутое/явное

## Ключевые выводы

1. **Системные промпты включают рассуждения**: Правильная конфигурация превращает LLM в рассуждающего агента
2. **Ограничения существуют**: Чистые LLM-рассуждения подвержены ошибкам на сложных задачах
3. **Инструменты помогают**: Внешние вычисления (калькуляторы и т.д.) повышают точность
4. **Итеративность важна**: Многошаговые паттерны рассуждений (такие как ReAct) работают лучше
5. **Прозраценность ценна**: Наблюдение за процессом рассуждений помогает отладке и верификации

## Следующие шаги

После понимания базовых рассуждений:
- **Добавьте инструменты**: Позвольте агенту использовать калькуляторы, базы данных, API
- **Реализуйте верификацию**: Проверяйте ответы, повторяйте при ошибках
- **Используйте цепочку рассуждений**: Сделайте рассуждения явными
- **Применяйте паттерн ReAct**: Систематически сочетайте рассуждения и использование инструментов

Этот пример является фундаментом для более сложных архитектур агентов, которые сочетают рассуждения с внешними возможностями.
