# Объяснение кода: simple-agent-with-memory.js

Этот пример расширяет простого агента **постоянной памятью**, позволяя ему запоминать информацию между сессиями при intelligentном избегании дублирующих сохранений.

## Ключевые компоненты

### 1. Импорт MemoryManager
```javascript
import {MemoryManager} from "./memory-manager.js";
```
Пользовательский класс для персистентности воспоминаний агента в JSON-файлах с единым хранилищем памяти.

### 2. Инициализация менеджера памяти
```javascript
const memoryManager = new MemoryManager('./agent-memory.json');
const memorySummary = await memoryManager.getMemorySummary();
```
- Загружает существующие воспоминания из файла
- Генерирует форматированную сводку для системного промпта
- Обрабатывает миграцию со старых схем памяти

### 3. Системный промпт с осведомлённостью о памяти и рассуждениями
```javascript
const systemPrompt = `
You are a helpful assistant with long-term memory.

Before calling any function, always follow this reasoning process:

1. **Compare** new user statements against existing memories below.
2. **If the same key and value already exist**, do NOT call saveMemory again.
   - Instead, simply acknowledge the known information.
   - Example: if the user says "My name is Malua" and memory already says "user_name: Malua", reply "Yes, I remember your name is Malua."
3. **If the user provides an updated value** (e.g., "I actually prefer sushi now"), 
   then call saveMemory once to update the value.
4. **Only call saveMemory for genuinely new information.**

When saving new data, call saveMemory with structured fields:
- type: "fact" or "preference"
- key: short descriptive identifier (e.g., "user_name", "favorite_food")
- value: the specific information (e.g., "Malua", "chinua")

Examples:
saveMemory({ type: "fact", key: "user_name", value: "Malua" })
saveMemory({ type: "preference", key: "favorite_food", value: "chinua" })

${memorySummary}
`;
```

**Что это делает:**
- Включает существующие воспоминания в промпт
- Предоставляет явные руководства по рассуждениям для предотвращения дублирующих сохранений
- Учит агента сравнивать перед сохранением
- Инструктирует, когда обновлять vs. подтверждать существующие данные

### 4. Функция saveMemory
```javascript
const saveMemory = defineChatSessionFunction({
    description: "Save important information to long-term memory (user preferences, facts, personal details)",
    params: {
        type: "object",
        properties: {
            type: {
                type: "string",
                enum: ["fact", "preference"]
            },
            key: { type: "string" },
            value: { type: "string" }
        },
        required: ["type", "key", "value"]
    },
    async handler({ type, key, value }) {
        await memoryManager.addMemory({ type, key, value });
        return `Memory saved: ${key} = ${value}`;
    }
});
```

**Что делает:**
- Использует структурированный формат ключ-значение для всех воспоминаний
- Сохраняет факты и предпочтения одним методом
- Автоматически обрабатывает дубликаты (обновляет при изменении значения)
- Персистит в JSON-файл
- Возвращает сообщение подтверждения

**Структура параметров:**
- `type`: Либо «fact», либо «preference»
- `key`: Короткий идентификатор (например, «user_name», «favorite_food»)
- `value`: Фактическая информация (например, «Alex», «pizza»)

### 5. Пример разговора
```javascript
const prompt1 = "Hi! My name is Alex and I love pizza.";
const response1 = await session.prompt(prompt1, {functions});
// Agent calls saveMemory twice:
// - saveMemory({ type: "fact", key: "user_name", value: "Alex" })
// - saveMemory({ type: "preference", key: "favorite_food", value: "pizza" })

const prompt2 = "What's my favorite food?";
const response2 = await session.prompt(prompt2, {functions});
// Agent recalls from memory: "Pizza"
```

## Как работает память

### Схема потока

```
Session 1:
User: "My name is Alex and I love pizza"
  ↓
Agent calls: saveMemory({ type: "fact", key: "user_name", value: "Alex" })
Agent calls: saveMemory({ type: "preference", key: "favorite_food", value: "pizza" })
  ↓
Saved to: agent-memory.json

Session 2 (after restart):
1. Load memories from agent-memory.json
2. Add to system prompt
3. Agent sees: "user_name: Alex" and "favorite_food: pizza"
4. Can use this information in responses

Session 3:
User: "My name is Alex"
  ↓
Agent compares: user_name already = "Alex"
  ↓
No function call! Just acknowledges: "Yes, I remember your name is Alex."
```

## Класс MemoryManager

Расположен в `memory-manager.js`:
```javascript
class MemoryManager {
  async loadMemories()           // Load from JSON (handles schema migration)
  async saveMemories()           // Write to JSON
  async addMemory()              // Unified method for all memory types
  async getMemorySummary()       // Format memories for system prompt
  extractKey()                   // Helper for migration
  extractValue()                 // Helper for migration
}
```

**Преимущества:**
- Единый метод для всех типов памяти
- Автоматическое обнаружение и предотвращение дубликатов
- Автоматическое обновление значений при изменении информации

## Ключевые концепции

### 1. Структурированный формат памяти
Все воспоминания теперь используют согласованную структуру:
```javascript
{
  type: "fact" | "preference",
  key: "user_name",           // Identifier
  value: "Alex",              // The actual data
  source: "user",             // Where it came from
  timestamp: "2025-10-29..."  // When it was saved/updated
}
```

### 2. Intelligentное предотвращение дубликатов
Агент обучен:
- **Сравнивать** перед сохранением
- **Пропускать** если данные идентичны
- **Обновлять** если значение изменилось
- **Подтверждать** существующие воспоминания вместо повторного сохранения

### 3. Персистентное состояние
- Воспоминания переживают перезапуск скриптов
- Хранятся в JSON-файле с метаданными
- Загружаются при запуске и инжектируются в промпт

### 4. Интеграция памяти в системный промпт
Воспоминания автоматически форматируются и инжектируются:
```
=== LONG-TERM MEMORY ===

Known Facts:
- user_name: Alex
- location: Paris

User Preferences:
- favorite_food: pizza
- preferred_language: French
```

## Почему это важно

**Без памяти:** Агент начинает с нуля каждый раз, задаёт одни и те же вопросы

**С базовой памятью:** Агент запоминает, но может тратить ресурсы на дубликаты

**С умной памятью:** Агент запоминает И избегает избыточных сохранений через рассуждения

Это обеспечивает:
- **Персонализированные ответы** на основе истории пользователя
- **Эффективное использование памяти** (без дублирующих записей)
- **Естественные разговоры**, которые кажутся непрерывными
- **Состоятельные агенты**, поддерживающие контекст
- **Автоматические обновления** при изменении информации

## Ожидаемый вывод

**Первый запуск:**
```
User: "Hi! My name is Alex and I love pizza."
AI: "Nice to meet you, Alex! I've noted that you love pizza."
[Calls saveMemory twice - new information saved]
```

**Второй запуск (после перезапуска):**
```
User: "What's my favorite food?"
AI: "Your favorite food is pizza! You mentioned that you love it."
[No function calls - recalls from loaded memory]
```

**Третий запуск (дублирующее утверждение):**
```
User: "My name is Alex."
AI: "Yes, I remember your name is Alex!"
[No function call - recognizes duplicate, just acknowledges]
```

**Четвёртый запуск (обновлённая информация):**
```
User: "I actually prefer sushi now."
AI: "Got it! I've updated your favorite food to sushi."
[Calls saveMemory once - updates existing value]
```

## Процесс рассуждений

Системный промпт явно направляет агента через это дерево решений:
```
New user statement
    ↓
Compare to existing memories
    ↓
    ├─→ Exact match? → Acknowledge only (no save)
    ├─→ Updated value? → Save to update
    └─→ New information? → Save as new
```

Этот подход «рассуждения сначала» делает агента более умным и эффективным при операциях с памятью!
