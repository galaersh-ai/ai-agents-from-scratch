# Объяснение кода: aot-agent.js

Этот пример демонстрирует паттерн промптинга **«Атом рассуждений»** с использованием математического калькулятора в качестве области.

## Трёхфазная архитектура

### Фаза 1: Планирование (LLM)
```javascript
async function generatePlan(userPrompt) {
    const grammar = await llama.createGrammarForJsonSchema(planSchema);
    const planText = await session.prompt(userPrompt, { grammar });
    return grammar.parse(planText);
}
```

**Ключевые моменты:**
- LLM выводит **структурированный JSON** (обеспечивается грамматикой)
- LLM НЕ выполняет вычисления
- Каждый атом представляет одну операцию
- Зависимости явные (массив `dependsOn`)

**Пример вывода:**
```json
{
  "atoms": [
    {"id": 1, "kind": "tool", "name": "add", "input": {"a": 15, "b": 7}},
    {"id": 2, "kind": "tool", "name": "multiply", "input": {"a": "<result_of_1>", "b": 3}},
    {"id": 3, "kind": "tool", "name": "subtract", "input": {"a": "<result_of_2>", "b": 10}},
    {"id": 4, "kind": "final", "name": "report", "dependsOn": [3]}
  ]
}
```

### Фаза 2: Валидация (Система)
```javascript
function validatePlan(plan) {
    const allowedTools = new Set(Object.keys(tools));
    
    for (const atom of plan.atoms) {
        if (ids.has(atom.id)) throw new Error(`Duplicate ID`);
        if (atom.kind === "tool" && !allowedTools.has(atom.name)) {
            throw new Error(`Unknown tool: ${atom.name}`);
        }
    }
}
```

**Валидирует:**
- Нет дублирующихся ID атомов
- Ссылаются только разрешённые инструменты
- Зависимости имеют смысл
- JSON-структура корректна

### Фаза 3: Выполнение (Система)
```javascript
function executePlan(plan) {
    const state = {};
    
    for (const atom of sortedAtoms) {
        // Resolve dependencies
        let resolvedInput = {};
        for (const [key, value] of Object.entries(atom.input)) {
            if (value.startsWith('<result_of_')) {
                const refId = parseInt(value.match(/\d+/)[0]);
                resolvedInput[key] = state[refId];
            }
        }
        
        // Execute
        state[atom.id] = tools[atom.name](resolvedInput.a, resolvedInput.b);
    }
}
```

**Ключевые поведения:**
- Выполняет атомы по порядку (отсортированные по ID)
- Разрешает ссылки `<result_of_N>` из состояния
- Каждый атом сохраняет свой результат в `state[atom.id]`
- Выполнение **детерминированное** (один и тот же план + одно и то же состояние = один и тот же результат)

## Почему это важно

### Сравнение с ReAct

| Аспект | ReAct | Атом рассуждений |
|--------|-------|------------------|
| **Планирование** | Неявное (в рассуждениях LLM) | Явное (JSON-структура) |
| **Выполнение** | LLM решает следующий шаг | Система следует плану |
| **Валидация** | Нет | До выполнения |
| **Отладка** | Сложно (трассировка через текст) | Легко (инспекция атомов) |
| **Тестирование** | Сложно (мок LLM) | Легко (тест исполнителя) |
| **Сбои** | Может галлюцинировать | Падает на конкретном атоме |

### Преимущества

1. **Нет скрытых рассуждений**: Каждая операция — явный атом
2. **Тестируемо**: Выполните план без участия LLM
3. **Отлаживаемо**: Точно знаете, какой атом упал
4. **Аудируемо**: План — это структура данных, а не текст
5. **Детерминированно**: Одинаковый ввод = одинаковый вывод (при одинаковом плане)

## Реализация инструментов

Инструменты — **чистые функции** без побочных эффектов:
```javascript
const tools = {
    add: (a, b) => {
        const result = a + b;
        console.log(`EXECUTING: add(${a}, ${b}) = ${result}`);
        return result;
    },
    // ... more tools
};
```

**Почему чистые функции?**
- Легко тестировать
- Легко воспроизводить
- Нет скрытого состояния
- Композитивны

## Поток состояния

```
User Question
      ↓
[LLM generates plan]
      ↓
{atoms: [...]} ← JSON plan
      ↓
[System validates]
      ↓
Plan valid
      ↓
[System executes atom 1] → state[1] = result
      ↓
[System executes atom 2] → state[2] = result (uses state[1])
      ↓
[System executes atom 3] → state[3] = result (uses state[2])
      ↓
Final Answer
```

## Обработка ошибок

```javascript
// Atom validation fails → re-prompt LLM
validatePlan(plan); // throws if invalid

// Tool execution fails → stop at that atom
if (b === 0) throw new Error("Division by zero");

// Dependency missing → clear error message
if (!(depId in state)) {
    throw new Error(`Atom ${atom.id} depends on incomplete atom ${depId}`);
}
```

## Когда использовать AoT

✅ **Используйте AoT когда:**
- Выполнение должно быть аудируемым
- Сбои должны быть восстанавливаемыми
- Несколько шагов с зависимостями
- Тестирование важно
- Важен комплаенс

❌ **Не используйте AoT когда:**
- Одноразовые задачи
- Творческие/исследовательские задачи
- Брейншторминг
- Естественный разговор

## Идеи расширения

1. **Добавьте атомы компенсации** для отката
2. **Добавьте логику повторов** для каждого атома
3. **Параллелизируйте независимые атомы** (атомы без общих зависимостей)
4. **Персистите план** для отладки
5. **Визуализируйте граф атомов** (дерево зависимостей)
