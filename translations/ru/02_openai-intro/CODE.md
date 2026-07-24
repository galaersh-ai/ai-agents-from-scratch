# Объяснение кода: OpenAI Intro

Это руководство разбирает каждый пример в `openai-intro.js`, объясняя, как работать с OpenAI API с нуля.

## Предварительные требования

Перед запуском этого примера Вам потребуется аккаунт OpenAI, API-ключ и действующий способ оплаты.

### Получение API-ключа

https://platform.openai.com/api-keys

### Добавление способа оплаты

https://platform.openai.com/settings/organization/billing/overview

### Настройка переменных окружения

```bash
   cp .env.example .env
```
Затем отредактируйте `.env` и добавьте Ваш реальный API-ключ.

## Установка и инициализация

```javascript
import OpenAI from 'openai';
import 'dotenv/config';

const client = new OpenAI({
    apiKey: process.env.OPENAI_API_KEY,
});
```

**Что происходит:**
- `import OpenAI from 'openai'` — Импорт официального SDK OpenAI для Node.js
- `import 'dotenv/config'` — Загрузка переменных окружения из файла `.env`
- `new OpenAI({...})` — Создание экземпляра клиента, который обрабатывает аутентификацию и запросы к API
- `process.env.OPENAI_API_KEY` — Ваш API-ключ с platform.openai.com (никогда не хардкодьте его!)

**Почему это важно:** Объект `client` — это Ваш интерфейс к моделям OpenAI. Все вызовы API проходят через этого клиента.

---

## Пример 1: Базовый Chat Completion

```javascript
const response = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: [
        { role: 'user', content: 'What is node-llama-cpp?' }
    ],
});

console.log(response.choices[0].message.content);
```

**Что происходит:**
- `chat.completions.create()` — Основной метод для отправки сообщений в модели ChatGPT
- `model: 'gpt-4o'` — Указывает, какую модель использовать (gpt-4o — последняя, самая способная модель)
- Массив `messages` — Содержит историю разговора
- `role: 'user'` — Указывает, что это сообщение от пользователя (от Вас)
- `response.choices[0]` — API возвращает массив возможных ответов; мы берём первый
- `message.content` — Текстовый ответ от AI

**Структура ответа:**
```javascript
{
  id: 'chatcmpl-...',
  object: 'chat.completion',
  created: 1234567890,
  model: 'gpt-4o',
  choices: [
    {
      index: 0,
      message: {
        role: 'assistant',
        content: 'node-llama-cpp is a...'
      },
      finish_reason: 'stop'
    }
  ],
  usage: {
    prompt_tokens: 10,
    completion_tokens: 50,
    total_tokens: 60
  }
}
```

---

## Пример 2: Системные промпты

```javascript
const response = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: [
        { role: 'system', content: 'You are a coding assistant that talks like a pirate.' },
        { role: 'user', content: 'Explain what async/await does in JavaScript.' }
    ],
});
```

**Что происходит:**
- `role: 'system'` — Специальный тип сообщения, который устанавливает поведение и личность AI
- Системные сообщения обрабатываются первыми и влияют на все последующие ответы
- Модель будет поддерживать это поведение на протяжении всего разговора

**Почему это важно:** Системные промпты — это способ специализировать поведение AI. Они являются фундаментом создания фокусированных агентов с конкретными ролями (переводчик, кодер, аналитик и т.д.).

**Ключевой вывод:** Одна и та же модель + разные системные промпты = полностью разные агенты!

---

## Пример 3: Контроль температуры

```javascript
// Focused response
const focusedResponse = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: [{ role: 'user', content: prompt }],
    temperature: 0.2,
});

// Creative response
const creativeResponse = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: [{ role: 'user', content: prompt }],
    temperature: 1.5,
});
```

**Что происходит:**
- `temperature` — Контролирует случайность вывода (диапазон: 0.0 до 2.0)
- **Низкая температура (0.0 - 0.3):**
    - Более фокусированный и детерминированный
    - Одинаковый ввод → похожий вывод
    - Лучше всего для: фактических ответов, генерации кода, извлечения данных
- **Средняя температура (0.7 - 1.0):**
    - Баланс креативности и согласованности
    - По умолчанию для большинства случаев использования
- **Высокая температура (1.2 - 2.0):**
    - Более креативный и разнообразный
    - Одинаковый ввод → очень разные выводы
    - Лучше всего для: творческого письма, брейншторминга, генерации историй

**Практическое использование:**
- Дописывание кода: температура 0.2
- Поддержка клиентов: температура 0.5
- Творческий контент: температура 1.2

---

## Пример 4: Контекст разговора

```javascript
const messages = [
    { role: 'system', content: 'You are a helpful coding tutor.' },
    { role: 'user', content: 'What is a Promise in JavaScript?' },
];

const response1 = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: messages,
});

// Add AI response to history
messages.push(response1.choices[0].message);

// Add follow-up question
messages.push({ role: 'user', content: 'Can you show me a simple example?' });

// Second request with full context
const response2 = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: messages,
});
```

**Что происходит:**
- Модели OpenAI **не имеют состояния** — они не помнят предыдущие разговоры
- Мы поддерживаем контекст, отправляя полную историю разговора с каждым запросом
- Каждый запрос независим; Вы должны включить все релевантные сообщения

**Порядок сообщений в массиве:**
1. Системный промпт (опционально, но рекомендуется первым)
2. Предыдущее сообщение пользователя
3. Предыдущий ответ ассистента
4. Текущее сообщение пользователя

**Почему это важно:** Вот так чат-боты запоминают контекст. Полный разговор отправляется каждый раз.

**Соображения производительности:**
- Больше сообщений = больше токенов = выше стоимость
- Длинные разговоры в итоге достигают лимитов токенов
- Реальным приложениям нужны стратегии обрезки или резюмирования разговора

---

## Пример 5: Стриминг ответов

```javascript
const stream = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: [
        { role: 'user', content: 'Write a haiku about programming.' }
    ],
    stream: true,  // Enable streaming
});

for await (const chunk of stream) {
    const content = chunk.choices[0]?.delta?.content || '';
    process.stdout.write(content);
}
```

**Что происходит:**
- `stream: true` — Вместо ожидания полного ответа получайте его по токену
- `for await...of` — Итерация по стриму по мере поступления чанков
- `delta.content` — Каждый чанк содержит небольшой кусочек текста (часто просто слово или часть слова)
- `process.stdout.write()` — Запись без перевода строки для постепенного отображения текста

**Стриминг vs. Не-стриминг:**

**Без стриминга (по умолчанию):**
```
[Request sent]
[Wait 5 seconds...]
[Full response arrives]
```

**Со стримингом:**
```
[Request sent]
Once [chunk arrives: "Once"]
upon [chunk arrives: " upon"]
a [chunk arrives: " a"]
time [chunk arrives: " time"]
...
```

**Почему это важно:**
- Лучший пользовательский опыт (немедленная обратная связь)
- Выглядит быстрее, хотя общее время аналогично
- Необходимо для чат-интерфейсов в реальном времени
- Позволяет раннюю обработку/отображение частичных результатов

**Когда использовать стриминг:**
- Интерактивные чат-приложения
- Генерация длинного контента
- Когда пользовательский опыт важнее простоты

**Когда НЕ использовать стриминг:**
- Простые скрипты или автоматизация
- Когда нужен полный ответ перед обработкой
- Пакетная обработка

---

## Пример 6: Использование токенов

```javascript
const response = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: [
        { role: 'user', content: 'Explain recursion in 3 sentences.' }
    ],
    max_tokens: 100,
});

console.log("Token usage:");
console.log("- Prompt tokens: " + response.usage.prompt_tokens);
console.log("- Completion tokens: " + response.usage.completion_tokens);
console.log("- Total tokens: " + response.usage.total_tokens);
```

**Что происходит:**
- `max_tokens` — Ограничивает длину ответа AI
- `response.usage` — Содержит детали потребления токенов
- **Prompt tokens:** Ваш ввод (сообщения, которые Вы отправили)
- **Completion tokens:** Вывод AI (ответ)
- **Total tokens:** Сумма обоих (за что Вы платите)

**Понимание токенов:**
- Токены ≠ слова
- 1 токен ≈ 0.75 слов (в английском)
- "hello" = 1 токен
- "chatbot" = 2 токена ("chat" + "bot")
- Пунктуация и пробелы считаются токенами

**Почему это важно:**
1. **Контроль стоимости:** Вы платите за токены
2. **Лимиты контекста:** Модели имеют максимальные лимиты токенов (например, gpt-4o: 128,000 токенов)
3. **Контроль ответа:** Используйте `max_tokens` для предотвращения слишком длинных ответов

**Практические лимиты:**
```javascript
// Prevent runaway responses
max_tokens: 150,  // ~100 words

// Brief responses
max_tokens: 50,   // ~35 words

// Longer content
max_tokens: 1000, // ~750 words
```

**Оценка стоимости (приблизительно):**
- GPT-4o: $5 за 1M входных токенов, $15 за 1M выходных токенов
- GPT-3.5-turbo: $0.50 за 1M входных токенов, $1.50 за 1M выходных токенов

---

## Пример 7: Сравнение моделей

```javascript
// GPT-4o - Most capable
const gpt4Response = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: [{ role: 'user', content: prompt }],
});

// GPT-3.5-turbo - Faster and cheaper
const gpt35Response = await client.chat.completions.create({
    model: 'gpt-3.5-turbo',
    messages: [{ role: 'user', content: prompt }],
});
```

**Доступные модели:**

| Модель | Лучше всего для | Скорость | Стоимость | Контекстное окно |
|--------|-----------------|----------|-----------|------------------|
| `gpt-4o` | Сложные задачи, рассуждения, точность | Средняя | $$$ | 128K токенов |
| `gpt-4o-mini` | Сбалансированная производительность/стоимость | Быстрая | $$ | 128K токенов |
| `gpt-3.5-turbo` | Простые задачи, высокий объём | Очень быстрая | $ | 16K токенов |

**Выбор правильной модели:**
- **Используйте GPT-4o когда:**
    - Требуются сложные рассуждения
    - Критична высокая точность
    - Работа с кодом или техническим контентом
    - Качество > скорость/стоимость

- **Используйте GPT-4o-mini когда:**
    - Нужна хорошая производительность при меньшей стоимости
    - Большинство универсальных задач

- **Используйте GPT-3.5-turbo когда:**
    - Простая классификация или извлечение
    - Высокообъёмные, низкосложностные задачи
    - Критична скорость
    - Ограничения бюджета

**Совет:** Начинайте с gpt-4o для разработки, затем оценивайте, подходят ли более дешёвые модели для Вашего случая использования.

---

## Обработка ошибок

```javascript
try {
    await basicCompletion();
} catch (error) {
    console.error("Error:", error.message);
    if (error.message.includes('API key')) {
        console.error("\nMake sure to set your OPENAI_API_KEY in a .env file");
    }
}
```

**Типичные ошибки:**
- `401 Unauthorized` — Неверный или отсутствующий API-ключ
- `429 Too Many Requests` — Превышен лимит частоты запросов
- `500 Internal Server Error` — Проблема с сервисом OpenAI
- `Context length exceeded` — Слишком много токенов в разговоре

**Лучшие практики:**
- Всегда используйте try-catch с async-вызовами
- Проверяйте типы ошибок и предоставляйте полезные сообщения
- Реализуйте логику повторов для временных сбоев
- Мониторьте использование токенов, чтобы избежать ошибок лимитов

---

## Ключевые выводы

1. **Без состояния:** Модели не помнят. Вы отправляете полный контекст каждый раз.
2. **Роли сообщений:** `system` (поведение), `user` (ввод), `assistant` (ответ AI)
3. **Температура:** Контролирует креативность (0 = фокусированная, 2 = креативная)
4. **Стриминг:** Лучший UX для приложений в реальном времени
5. **Управление токенами:** Мониторьте использование для стоимости и лимитов
6. **Выбор модели:** Выбирайте на основе сложности задачи и бюджета
