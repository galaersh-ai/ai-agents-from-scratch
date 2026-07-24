# Объяснение кода: react-agent.js

Этот пример реализует **паттерн ReAct** (Рассуждение + Действие) — мощный подход для многошагового решения задач с инструментами.

## Что такое ReAct?

ReAct = **Rea**soning (Рассуждение) + **Act**ing (Действие)

Агент чередует:
1. **Мышление** (рассуждение о том, что делать)
2. **Действие** (использование инструментов)
3. **Наблюдение** (наблюдение за результатами инструментов)
4. Повтор до решения задачи

## Ключевые компоненты

### 1. Системный промпт ReAct (строки 20-52)
```javascript
const systemPrompt = `You are a mathematical assistant that uses the ReAct approach.

CRITICAL: You must follow this EXACT pattern:

Thought: [Explain what calculation you need]
Action: [Call ONE tool]
Observation: [Wait for result]
Thought: [Analyze result]
Action: [Call another tool if needed]
...
Thought: [Once you have all information]
Answer: [Final answer and STOP]
```

**Ключевые инструкции:**
- Явный пошаговый паттерн
- Один вызов инструмента за раз
- Продолжайте до финального ответа
- Остановитесь после «Answer:»

### 2. Инструменты-калькуляторы (строки 60-159)

Четыре базовые математические операции:
```javascript
const add = defineChatSessionFunction({...});
const multiply = defineChatSessionFunction({...});
const subtract = defineChatSessionFunction({...});
const divide = defineChatSessionFunction({...});
```

Каждый инструмент:
- Принимает два числа (a, b)
- Выполняет операцию
- Логирует вызов
- Возвращает результат как строку

### 3. Цикл агента ReAct (строки 164-212)

```javascript
async function reactAgent(userPrompt, maxIterations = 10) {
    let iteration = 0;
    let fullResponse = "";
    
    while (iteration < maxIterations) {
        iteration++;
        
        // Prompt the LLM
        const response = await session.prompt(
            iteration === 1 ? userPrompt : "Continue your reasoning.",
            {
                functions,
                maxTokens: 300,
                onTextChunk: (chunk) => {
                    process.stdout.write(chunk);  // Stream output
                    currentChunk += chunk;
                }
            }
        );
        
        fullResponse += currentChunk;
        
        // Check if final answer reached
        if (response.toLowerCase().includes("answer:")) {
            return fullResponse;
        }
    }
}
```

**Как это работает:**
1. Цикл до maxIterations итераций
2. На первой итерации: отправляем вопрос пользователя
3. На последующих итерациях: просим продолжить
4. Стримим вывод в реальном времени
5. Останавливаемся, когда появляется «Answer:»
6. Возвращаем полный след рассуждений

### 4. Пример запроса (строки 215-220)

```javascript
const queries = [
    "A store sells 15 items Monday at $8 each, 20 items Tuesday at $8 each, 
     10 items Wednesday at $8 each. What's the average items per day and total revenue?"
];
```

Сложная задача, требующая нескольких вычислений:
- 15 × 8
- 20 × 8
- 10 × 8
- Суммирование результатов
- Вычисление среднего
- Форматирование ответа

## Поток ReAct

### Пример выполнения

```
USER: "A store sells 15 items at $8 each and 20 items at $8 each. Total revenue?"

Iteration 1:
Thought: First I need to calculate 15 × 8
Action: multiply(15, 8)
Observation: 120

Iteration 2:
Thought: Now I need to calculate 20 × 8
Action: multiply(20, 8)
Observation: 160

Iteration 3:
Thought: Now I need to add both results
Action: add(120, 160)
Observation: 280

Iteration 4:
Thought: I have the total revenue
Answer: The total revenue is $280
```

**Цикл останавливается**, потому что обнаружен «Answer:».

## Почему ReAct работает

### Традиционный подход (Проваливается)
```
User: "Complex math problem"
LLM: [Tries to calculate in head]
→ Often wrong due to arithmetic errors
```

### Подход ReAct (Успешен)
```
User: "Complex math problem"
LLM: "I need to calculate X"
  → Calls calculator tool
  → Gets accurate result
  → Uses result for next step
  → Continues until solved
```

## Ключевые концепции

### 1. Явные рассуждения
Агент должен «показывать свою работу»:
```
Thought: What do I need to do?
Action: Do it
Observation: What happened?
```

### 2. Использование инструментов на каждом шаге

```
Don't calculate: 15 × 8 = 120 (may be wrong)
Do calculate: multiply(15, 8) → 120 (always correct)
```

### 3. Итеративное решение задач

```
Complex Problem → Break into steps → Solve each step → Combine results
```

### 4. Самокоррекция
Агент может замечать плохие результаты и пробовать снова:
```
Thought: That doesn't look right
Action: Let me recalculate
```

## Отладочный вывод

Код включает PromptDebugger (строки 228-234):
```javascript
const promptDebugger = new PromptDebugger({
    outputDir: './logs',
    filename: 'react_calculator.txt',
    includeTimestamp: true
});
await promptDebugger.debugContextState({session, model});
```

Сохраняет полную историю промптов в логи для отладки.

## Ожидаемый вывод

```
========================================================
USER QUESTION: [Problem statement]
========================================================

--- Iteration 1 ---
Thought: First I need to multiply 15 by 8
Action: multiply(15, 8)

   🔧 TOOL CALLED: multiply(15, 8)
   📊 RESULT: 120

Observation: 120

--- Iteration 2 ---
Thought: Now I need to multiply 20 by 8
Action: multiply(20, 8)

   🔧 TOOL CALLED: multiply(20, 8)
   📊 RESULT: 160

... continues ...

--- Iteration N ---
Thought: I have all the information
Answer: [Final answer]

========================================================
FINAL ANSWER REACHED
========================================================
```

## Почему это важно

### Обеспечивает сложные задачи
- Многошаговые рассуждения
- Точные вычисления
- Самокоррекция
- Прозрачный процесс

### Фундамент современных агентов
Этот паттерн обеспечивает:
- Агентов LangChain
- AutoGPT
- BabyAGI
- Большинство production-фреймворков агентов

### Наблюдаемые рассуждения
В отличие от LLM «чёрного ящика», Вы видите:
- О чём думает агент
- Какие инструменты он использует
- Почему он принимает решения
- Где он может потерпеть неудачу

## Лучшие практики

1. **Ясный системный промпт**: Определите точный паттерн
2. **Один инструмент за действие**: Не комбинируйте операции
3. **Ограничивайте итерации**: Предотвращайте бесконечные циклы
4. **Стримим вывод**: Показываем прогресс
5. **Тщательно отлаживаемся**: Используйте PromptDebugger

## Сравнение

```
Simple Agent vs ReAct Agent
────────────────────────────
Single prompt/response      Multi-step iteration
One tool call (maybe)       Multiple tool calls
No visible reasoning        Explicit reasoning
Works for simple tasks      Handles complex problems
```

Это передовой паттерн для создания способных AI-агентов!
