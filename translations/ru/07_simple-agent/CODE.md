# Объяснение кода: simple-agent.js

Этот файл демонстрирует **вызов функций** — ключевую фичу, которая превращает LLM из генератора текста в агента, способного выполнять действия с помощью инструментов.

## Пошаговый разбор кода

### 1. Импорт и настройка (строки 1-7)
```javascript
import {defineChatSessionFunction, getLlama, LlamaChatSession} from "node-llama-cpp";
import {fileURLToPath} from "url";
import path from "path";
import {PromptDebugger} from "../helper/prompt-debugger.js";

const __dirname = path.dirname(fileURLToPath(import.meta.url));
const debug = false;
```
- **defineChatSessionFunction**: Ключевой импорт для создания вызываемых функций
- **PromptDebugger**: Утилита для отладки промптов (рассматривается в конце)
- **debug**: Контролирует подробное логирование

### 2. Инициализация и загрузка модели (строки 9-17)
```javascript
const llama = await getLlama({debug});
const model = await llama.loadModel({
    modelPath: path.join(
        __dirname,
        "../",
        "models",
        "Qwen3-1.7B-Q8_0.gguf"
    )
});
const context = await model.createContext({contextSize: 2000});
```
- Использует модель Qwen3-1.7B (хороша для вызова функций)
- Явно устанавливает размер контекста в 2000 токенов

### 3. Системный промпт для конвертации времени (строки 20-23)
```javascript
const systemPrompt = `You are a professional chronologist who standardizes time representations across different systems.
    
Always convert times from 12-hour format (e.g., "1:46:36 PM") to 24-hour format (e.g., "13:46") without seconds 
before returning them.`;
```

**Назначение:**
- Определяет роль и поведение агента
- Инструктирует по формату вывода (24-часовой, без секунд)
- Обеспечивает согласованность представления времени

### 4. Создание сессии (строки 25-28)
```javascript
const session = new LlamaChatSession({
    contextSequence: context.getSequence(),
    systemPrompt,
});
```
Стандартная сессия с системным промптом.

### 5. Определение функции-инструмента (строки 30-39)
```javascript
const getCurrentTime = defineChatSessionFunction({
    description: "Get the current time",
    params: {
        type: "object",
        properties: {}
    },
    async handler() {
        return new Date().toLocaleTimeString();
    }
});
```

**Разбор по частям:**

**description:**
- Говорит LLM, что делает эта функция
- LLM читает это, чтобы решить, когда её вызвать

**params:**
- Определяет параметры функции (формат JSON Schema)
- Пустые `properties: {}` означают, что параметры не нужны
- Тип должен быть «object», даже если нет свойств

**handler:**
- Фактическая JavaScript-функция, которая выполняется
- Возвращает текущее время как строку (например, «1:46:36 PM»)
- Может быть async (используйте await внутри)

### Как работает вызов функций

```
1. User asks: "What time is it?"
2. LLM reads: 
   - System prompt
   - Available functions (getCurrentTime)
   - Function description
3. LLM decides: "I should call getCurrentTime()"
4. Library executes: handler()
5. Handler returns: "1:46:36 PM"
6. LLM receives result as "tool output"
7. LLM processes: Converts to 24-hour format per system prompt
8. LLM responds: "13:46"
```

### 6. Регистрация функций (строка 41)
```javascript
const functions = {getCurrentTime};
```
- Создаёт объект со всеми доступными функциями
- Несколько функций: `{getCurrentTime, getWeather, calculate, ...}`
- LLM может выбрать, какие функции вызвать

### 7. Определение пользовательского промпта (строка 42)
```javascript
const prompt = `What time is it right now?`;
```
Вопрос, требующий использования инструмента.

### 8. Выполнение с функциями (строка 45)
```javascript
const a1 = await session.prompt(prompt, {functions});
console.log("AI: " + a1);
```
- **{functions}** делает инструменты доступными для LLM
- LLM автоматически вызовет getCurrentTime, если нужно
- Ответ включает результат инструмента, обработанный LLM

### 9. Отладка контекста промпта (строки 49-55)
```javascript
const promptDebugger = new PromptDebugger({
    outputDir: './logs',
    filename: 'qwen_prompts.txt',
    includeTimestamp: true,
    appendMode: false
});
await promptDebugger.debugContextState({session, model});
```

**Что это делает:**
- Сохраняет весь промпт, отправленный модели
- Показывает точно то, что видит LLM (включая определения функций)
- Полезно для отладки, почему модель вызывает/не вызывает функции
- Записывает в `./logs/qwen_prompts_[timestamp].txt`

### 10. Очистка (строки 58-61)
```javascript
session.dispose()
context.dispose()
model.dispose()
llama.dispose()
```
Стандартная очистка.

## Продемонстрированные ключевые концепции

### 1. Вызов функций (использование инструментов)

Вот что делает это «агентом»:
```
Without tools:          With tools:
LLM → Text only        LLM → Can take actions
                              ↓
                       Call functions
                       Access data
                       Execute code
```

### 2. Паттерн определения функции

```javascript
defineChatSessionFunction({
    description: "What the function does",  // LLM reads this
    params: {                               // Expected parameters
        type: "object",
        properties: {
            paramName: {
                type: "string",
                description: "What this param is for"
            }
        },
        required: ["paramName"]
    },
    handler: async (params) => {            // Your code
        // Do something with params
        return result;
    }
});
```

### 3. JSON Schema для параметров

Используется стандартный JSON Schema:
```javascript
// No parameters
properties: {}

// One string parameter
properties: {
    city: {
        type: "string",
        description: "City name"
    }
}

// Multiple parameters
properties: {
    a: { type: "number" },
    b: { type: "number" }
},
required: ["a", "b"]
```

### 4. Принятие решений агентом

```
User: "What time is it?"
         ↓
    LLM thinks:
    "I need current time"
    "I see function: getCurrentTime"
    "Description matches what I need"
         ↓
    LLM outputs special format:
    {function_call: "getCurrentTime"}
         ↓
    Library intercepts and runs handler()
         ↓
    Handler returns: "1:46:36 PM"
         ↓
    LLM receives: Tool result
         ↓
    LLM applies system prompt:
    Convert to 24-hour format
         ↓
    Final answer: "13:46"
```

## Случаи использования

### 1. Получение информации
```javascript
const getWeather = defineChatSessionFunction({
    description: "Get weather for a city",
    params: {
        type: "object",
        properties: {
            city: { type: "string" }
        }
    },
    handler: async ({city}) => {
        return await fetchWeather(city);
    }
});
```

### 2. Вычисления
```javascript
const calculate = defineChatSessionFunction({
    description: "Perform arithmetic calculation",
    params: {
        type: "object",
        properties: {
            expression: { type: "string" }
        }
    },
    handler: async ({expression}) => {
        return eval(expression); // (Be careful with eval!)
    }
});
```

### 3. Доступ к данным
```javascript
const queryDatabase = defineChatSessionFunction({
    description: "Query user database",
    params: {
        type: "object",
        properties: {
            userId: { type: "string" }
        }
    },
    handler: async ({userId}) => {
        return await db.users.findById(userId);
    }
});
```

### 4. Внешние API
```javascript
const searchWeb = defineChatSessionFunction({
    description: "Search the web",
    params: {
        type: "object",
        properties: {
            query: { type: "string" }
        }
    },
    handler: async ({query}) => {
        return await googleSearch(query);
    }
});
```

## Ожидаемый вывод

При запуске:
```
AI: 13:46
```

LLM:
1. Вызвала getCurrentTime() внутренне
2. Получила «1:46:36 PM»
3. Конвертировала в 24-часовой формат
4. Убрала секунды
5. Вернула «13:46»

## Отладка с PromptDebugger

Отладочный вывод показывает полный промпт, включая схемы функций:
```
System: You are a professional chronologist...

Functions available:
- getCurrentTime: Get the current time
  Parameters: (none)

User: What time is it right now?
```

Это помогает отладить:
- Видела ли модель функцию?
- Было ли описание понятным?
- Соответствовали ли параметры ожиданиям?

## Почему это важно для AI-агентов

### Агенты = LLM + Инструменты

```
LLM alone:                    LLM + Tools:
├─ Generate text              ├─ Generate text
└─ That's it                  ├─ Access real data
                              ├─ Perform calculations
                              ├─ Call APIs
                              ├─ Execute actions
                              └─ Interact with world
```

### Фундамент для сложных агентов

Этот простой пример является фундаментом для:
- **Исследовательских агентов**: Веб-поиск, чтение документов
- **Кодирующих агентов**: Запуск кода, проверка ошибок
- **Персональных ассистентов**: Календарь, почта, напоминания
- **Аналитических агентов**: Запросы к базам данных, вычисление статистики

Всё начинается с базового вызова функций!

## Лучшие практики

1. **Ясные описания**: LLM использует их для решения, когда вызывать
2. **Типизация**: Правильно используйте JSON Schema
3. **Обработка ошибок**: Handler должен ловить ошибки
4. **Возвращайте строки**: LLM лучше всего обрабатывает текст
5. **Держите функции фокусированными**: Одна чёткая цель на функцию

Это минимально жизнеспособный агент: одна LLM + один инструмент + правильная конфигурация.
