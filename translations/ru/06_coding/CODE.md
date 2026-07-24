# Объяснение кода: coding.js

Этот файл демонстрирует **стриминг ответов** с лимитами токенов и выводом в реальном времени, показывая, как получить немедленную обратную связь от LLM по мере генерации текста.

## Пошаговый разбор кода

### 1. Импорт и настройка (строки 1-8)
```javascript
import {
    getLlama,
    HarmonyChatWrapper,
    LlamaChatSession,
} from "node-llama-cpp";
import {fileURLToPath} from "url";
import path from "path";

const __dirname = path.dirname(fileURLToPath(import.meta.url));
```
- Стандартная настройка для взаимодействия с LLM
- **HarmonyChatWrapper**: Обёртка формата чата для моделей, использующих формат Harmony (подробнее ниже)

### 2. Понимание формата Harmony Chat

#### Что такое Harmony?
Harmony — это структурированный формат сообщений для чат-взаимодействий с несколькими ролями, разработанный OpenAI для моделей gpt-oss. Это не просто формат промпта — это полное переосмысление того, как модели должны структурировать свои выводы, особенно для сложных рассуждений и использования инструментов.

#### Структура формата Harmony

Формат использует специальные токены и синтаксис для определения ролей: `system`, `developer`, `user`, `assistant` и `tool`, а также «каналы» вывода (`analysis`, `commentary`, `final`), которые позволяют модели рассуждать внутренне, вызывать инструменты и генерировать чистые ответы для пользователя.

**Базовая структура сообщения:**
```
<|start|>ROLE<|message|>CONTENT<|end|>
<|start|>assistant<|channel|>CHANNEL<|message|>CONTENT<|end|>
```

**Пять ролей в порядке иерархии** (system > developer > user > assistant > tool):

1. **system**: Глобальная идентичность, ограничения и конфигурация модели
2. **developer**: Политика продукта и инструкции по стилю (то, что обычно считается «системным промптом»)
3. **user**: Сообщения и запросы пользователя
4. **assistant**: Ответы модели
5. **tool**: Результаты выполнения инструментов

**Три канала вывода:**

1. **analysis**: Приватные рассуждения (цепочка мыслей), не показываемые пользователям
2. **commentary**: Преамбулы вызова инструментов и обновления процесса
3. **final**: Чистые ответы для пользователя

**Пример Harmony в действии:**
```
<|start|>system<|message|>You are a helpful assistant.<|end|>
<|start|>developer<|message|>Always be concise.<|end|>
<|start|>user<|message|>What time is it?<|end|>
<|start|>assistant<|channel|>commentary<|message|>{"tool_use": {"name": "get_current_time", "arguments": {}}}<|end|>
<|start|>tool<|message|>{"time": "2025-10-25T13:47:00Z"}<|end|>
<|start|>assistant<|channel|>final<|message|>The current time is 1:47 PM UTC.<|end|>
```

#### Зачем использовать Harmony?

Harmony отделяет то, как модель думает, какие действия она предпринимает и что в итоге идёт пользователю. Это приводит к более чистому использованию инструментов, безопасным дефолтам для UI и лучшей наблюдаемости. Для нашего примера перевода:

- Канал `final` гарантирует, что мы получим только перевод, без объяснений
- Структурированный формат помогает модели более надёжно следовать инструкциям
- Иерархия ролей предотвращает конфликты инструкций

**Важное примечание**: Модели должны быть специально обучены или дообучены для корректного генерирования вывода Harmony. Нельзя просто применить этот формат к любой модели. Apertus и другие модели, не обученные на Harmony, могут быть смущены этой структурой, но HarmonyChatWrapper в node-llama-cpp автоматически обрабатывает необходимое форматирование.

### 3. Загрузка модели (строки 10-18)
```javascript
const llama = await getLlama();
const model = await llama.loadModel({
    modelPath: path.join(
        __dirname,
        "../",
        "models",
        "hf_giladgd_gpt-oss-20b.MXFP4.gguf"
    )
});
```
- Использует **gpt-oss-20b**: модель с 20 миллиардами параметров
- **MXFP4**: Смешанная 4-битная квантизация для меньшего размера
- Более крупная модель = лучшие объяснения кода

### 4. Создание контекста и сессии (строки 19-22)
```javascript
const context = await model.createContext();
const session = new LlamaChatSession({
    chatWrapper: new HarmonyChatWrapper(),
    contextSequence: context.getSequence(),
});
```
Базовая настройка сессии без системного промпта.

### 5. Определение вопроса (строка 24)
```javascript
const q1 = `What is hoisting in JavaScript? Explain with examples.`;
```
Технический вопрос по программированию, требующий детального объяснения.

### 6. Отображение размера контекста (строка 26)
```javascript
console.log('context.contextSize', context.contextSize)
```
- Показывает максимальный размер контекстного окна
- Помогает понять ограничения памяти
- Полезно для отладки

### 7. Стриминговый выполнение промпта (строки 28-36)
```javascript
const a1 = await session.prompt(q1, {
    // Tip: let the lib choose or cap reasonably; using the whole context size can be wasteful
    maxTokens: 2000,

    // Fires as soon as the first characters arrive
    onTextChunk: (text) => {
        process.stdout.write(text); // optional: live print
    },
});
```

**Ключевые параметры:**

**maxTokens: 2000**
- Ограничивает длину ответа 2000 токенов (~1500 слов)
- Предотвращает бесконечную генерацию
- Экономит время и вычислительные ресурсы
- Без лимита: модель использует весь контекст

**Колбэк onTextChunk**
- Срабатывает **при генерации каждого токена**
- Получает текст по мере его создания
- `process.stdout.write()`: Печатает без переводов строки
- Создаёт эффект печати в реальном времени

### Как работает стриминг

```
Without streaming:
User → [Wait 10 seconds...] → Complete response appears

With streaming:
User → [Token 1] → [Token 2] → [Token 3] → ... → Complete
       "What"      "is"        "hoisting"
       (Immediate feedback!)
```

### 8. Отображение финального ответа (строка 38)
```javascript
console.log("\n\nFinal answer:\n", a1);
```
- Печатает полный ответ ещё раз
- Полезно для логирования или верификации
- Показывает полный текст после стриминга

### 9. Очистка (строки 41-44)
```javascript
session.dispose()
context.dispose()
model.dispose()
llama.dispose()
```
Стандартная очистка ресурсов.

## Продемонстрированные ключевые концепции

### 1. Стриминг ответов

**Почему стриминг важен:**
- **Лучший UX**: Пользователи видят прогресс немедленно
- **Ранняя остановка**: Можно остановить, если ответ идёт не по плану
- **Воспринимаемая скорость**: Выглядит быстрее ожидания
- **Отладка**: Наблюдайте за генерацией в реальном времени

**Сравнение:**
```
Non-streaming:           Streaming:
═══════════════         ═══════════════
Request sent            Request sent
[10s wait...]           "What" (0.1s)
Complete response       "is" (0.2s)
                        "hoisting" (0.3s)
                        ... continues
                        (Same total time, better experience!)
```

### 2. Лимиты токенов

**maxTokens контролирует длину генерации:**

```
No limit:               With limit (2000):
─────────             ─────────────────
May generate forever   Stops at 2000 tokens
Uses entire context    Saves computation
Unpredictable cost     Predictable cost
```

**Приблизительный подсчёт токенов:**
- 1 токен ≈ 0.75 слов (на английском)
- 2000 токенов ≈ 1500 слов
- 4-5 абзацев детального объяснения

### 3. Паттерн обратной связи в реальном времени

Колбэк `onTextChunk` обеспечивает:
```javascript
onTextChunk: (text) => {
    // Do anything with each chunk:
    process.stdout.write(text);      // Console output
    // socket.emit('chunk', text);   // WebSocket to client
    // buffer += text;               // Accumulate for processing
    // analyzePartial(text);         // Real-time analysis
}
```

### 4. Осведомлённость о размере контекста

```javascript
console.log('context.contextSize', context.contextSize)
```

Показывает объём памяти модели:
- Маленькие модели: 2048-4096 токенов
- Средние модели: 8192-16384 токенов
- Большие модели: 32768+ токенов

**Почему это важно:**
```
Context Size: 4096 tokens
Prompt: 100 tokens
Max response: 2000 tokens
History: Up to 1996 tokens
```

## Случаи использования

### 1. Объяснение кода (этот пример)
```javascript
prompt: "Explain hoisting in JavaScript"
→ Streams detailed explanation with examples
```

### 2. Генерация длинного контента
```javascript
prompt: "Write a blog post about AI agents"
maxTokens: 3000
→ Streams article as it's written
```

### 3. Интерактивное наставничество
```javascript
// User sees explanation being built
prompt: "Teach me about closures"
onTextChunk: (text) => displayToUser(text)
```

### 4. Веб-приложения
```javascript
// Server-Sent Events or WebSocket
onTextChunk: (text) => {
    websocket.send(text);  // Send to browser
}
```

## Соображения по производительности

### Скорость генерации токенов

Зависит от:
- **Размера модели**: Больше = медленнее за токен
- **Оборудования**: GPU > CPU
- **Квантизации**: Меньше бит = быстрее
- **Длины контекста**: Длиннее контекст = медленнее

**Типичные скорости:**
```
Model Size    GPU (RTX 4090)    CPU (M2 Max)
──────────    ──────────────    ────────────
1.7B          50-80 tok/s       15-25 tok/s
8B            20-35 tok/s       5-10 tok/s
20B           10-15 tok/s       2-4 tok/s
```

### Когда использовать maxTokens

```
✓ Use maxTokens when:
  • Response length is predictable
  • You want to save computation
  • Testing/debugging
  • API rate limiting

✗ Don't limit when:
  • Need complete answer
  • Length varies greatly
  • Using stop sequences instead
```

## Продвинутые паттерны стриминга

### Паттерн 1: Прогрессивное улучшение
```javascript
let buffer = '';
onTextChunk: (text) => {
    buffer += text;
    if (buffer.includes('\n\n')) {
        // Complete paragraph ready
        processParagraph(buffer);
        buffer = '';
    }
}
```

### Паттерн 2: Ранняя остановка
```javascript
let isRelevant = true;
onTextChunk: (text) => {
    if (text.includes('irrelevant_keyword')) {
        isRelevant = false;
        // Stop generation (would need additional API)
    }
}
```

### Паттерн 3: Мульти-потребитель
```javascript
onTextChunk: (text) => {
    console.log(text);           // Console
    logFile.write(text);         // File
    websocket.send(text);        // Client
    analyzer.process(text);      // Analysis
}
```

## Ожидаемый вывод

При запуске Вы увидите:
1. Размер контекста в логе (например, «context.contextSize 32768»)
2. Стриминговый ответ, появляющийся по токену
3. Полный финальный ответ, напечатанный снова

Пример вывода:
```
context.contextSize 32768
Hoisting is a JavaScript mechanism where variables and function 
declarations are moved to the top of their scope before code 
execution. For example:

console.log(x); // undefined (not an error!)
var x = 5;

This works because...
[continues streaming...]

Final answer:
[Complete response printed again]
```

## Почему это важно для AI-агентов

### Пользовательский опыт
- Агенты реального времени выглядят более отзывчивыми
- Пользователи могут прервать, если агент идёт не в том направлении
- Лучше подходит для разговорных интерфейсов

### Управление ресурсами
- Лимиты токенов предотвращают бесконечную генерацию
- Предсказуемые стоимость и тайминги
- Можно рано отменить дорогие операции

### Паттерны интеграции
- Веб-UI показывают эффект «печати»
- CLI отображают прогрессивный вывод
- API эффективно стримят клиентам

Этот паттерн необходим для production-систем агентов, где важны пользовательский опыт и контроль ресурсов.
