# Объяснение кода: error-handling.js

Этот файл демонстрирует **комплексную обработку ошибок для программ в стиле агентов**: типизированную таксономию ошибок, таймауты и повторы с backoff, симуляцию отказов инструментов, **деградированный режим** при сбое пути LLM и **`AgentWorkflowError`** при нарушении оркестрации. Он запускается локально с **`node-llama-cpp`** и моделью GGUF (тот же стек, что и в других примерах агентов).

**Запуск:** `node examples/11_error-handling/error-handling.js`

---

## Пошаговый разбор кода

### 1. Импорты

```javascript
import crypto from "node:crypto";
import { defineChatSessionFunction, getLlama, LlamaChatSession } from "node-llama-cpp";
import { fileURLToPath } from "url";
import path from "path";
```

**Что происходит:**
- **`crypto`** — UUID для correlation ID (`randomUUID`) и jitter для задержек повторов (`randomInt`).
- **`node-llama-cpp`** — Загрузка модели, чат-сессии и **`defineChatSessionFunction`** для инструментов.
- **`url` / `path`** — Разрешение **`__dirname`** в ES-модуле и построение путей к файлу **`.gguf`**.

---

### 2. Таксономия ошибок

```javascript
class AppError extends Error { /* code, userMessage, retryable, details, cause */ }
class ValidationError extends AppError { /* VALIDATION_ERROR */ }
class LLMCallError extends AppError { /* LLM_CALL_FAILED; optional model */ }
class ToolExecutionError extends AppError { /* TOOL_EXECUTION_FAILED; toolName */ }
class AgentWorkflowError extends AppError { /* AGENT_WORKFLOW_FAILED; step */ }
```

**Назначение:**
- **`AppError`** — Одна структура для логов, повторов и текста для пользователей: стабильный **`code`**, безопасный **`userMessage`**, структурированные **`details`**, опциональный **`cause`**.
- Подклассы устанавливают разумные дефолты (например, валидация не повторяется; инструменты часто не повторяются, если Вы не передаёте **`retryable: true`**).
- **`AgentWorkflowError`** добавляет **`step`** (например, `policy_guard`, `resolve_user_profile`) для ошибок на уровне оркестрации. Комментарий в исходном коде объясняет, как один этот класс может представлять ошибки стилей política / рабочий процесс / система в маленькой демонстрации.

---

### 3. `sleep`

Простая задержка на основе `Promise`. Используется **`withRetries`** между попытками и внутри фейковых «сетевых» инструментов.

---

### 4. `withTimeout`

**`Promise.race`** между реальной работой и таймером. При таймауте отклоняет с **`AppError`** с кодом **`TIMEOUT`**, **`retryable: true`** и **`details: { label, ms }`**. Таймер очищается в **`finally`**.

**Почему это важно:** Каждый вызов LLM или инструмента, который может зависнуть, должен быть ограничен, чтобы агент мог восстановиться вместо бесконечного зависания.

---

### 5. `normalizeUnknownError`

Если брошенное значение уже является **`AppError`**, возвращает его. В противном случае оборачивает как **`UNKNOWN_ERROR`** (не повторяется), сохраняет исходное имя/сообщение в **`details`**, устанавливает **`cause`** как исходную ошибку.

**Почему это важно:** Блоки catch часто получают **`Error`**, строки или специфичные для библиотеки типы; нормализация делает **`retryOn`** и **`formatUserFacingError`** предсказуемыми.

---

### 6. `classifyError`

Вызывает **`normalizeUnknownError`**, затем возвращает **`{ error, retryable, type }`**, где **`type`** — это **`error.code`**.

**Почему это важно:** Одно место для решения «повторять?» вместо повторения проверок **`instanceof`** в предикатах **`promptLLM`**, **`runAgent`** и **`withRetries`**.

---

### 7. `isRetryable`

Возвращает **`classifyError(err).retryable`**. Используется как **дефолтный** **`retryOn`** для **`withRetries`**.

---

### 8. `jitteredBackoffDelay`

Экспоненциальная задержка с ограничением **`maxDelayMs`**, плюс **случайный jitter** через **`crypto.randomInt`**, чтобы многие клиенты не повторялись синхронно.

---

### 9. `withRetries`

Запускает **`fn`** до **`retries + 1`** раз. После падения, если есть оставшиеся попытки и **`retryOn(err)`** истинно, ждёт (**`sleep`** + **`jitteredBackoffDelay`**), логирует **`[retry]`** и повторяет. В противном случае бросает **`lastErr`**.

---

### 10. `formatUserFacingError`

Строит строку, показываемую «пользователю» в демо: **`userMessage`** плюс **`(Reference: <correlationId>)`**, или общий fallback, если ошибка не была **`AppError`**.

---

### 11. `printAgentWorkflowErrorBanner`

Когда ловится **`AgentWorkflowError`**, печатает ограниченный блок в **stderr**: шаг, код, correlation ID, сообщения, **`details`** и краткую сводку **`cause`**. Дополняет однострочный JSON-лог **`[agent_error]`**.

---

### 12. `SIMULATION` и фейковые инструменты

```javascript
const SIMULATION = {
  forceNotFound: new Set(["u_999"]),
  forcePrimaryAndFallbackFail: new Set(["u_777"]),
};
```

**`fetchUserFromPrimary`** — Симулирует задержку; **`u_999`** > не повторяется «not found»; **`u_777`** > всегда повторяется падение основного пути (демо); иначе ~20% случайного временного падения; успех возвращает профиль с **`source: "primary"`**.

**`fetchUserFromFallback`** — Профиль с меньшей детализацией; для **`u_777`** бросает исключение, чтобы цепочка **основной > запасной** могла детерминированно выдать **`AgentWorkflowError`**.

---

### 13. Инициализация модели и сессии

Аналогичный паттерн **`simple-agent.js`**: **`getLlama`**, **`loadModel`** (путь к **`models/Qwen3-1.7B-Q8_0.gguf`**), **`createContext`**, **`LlamaChatSession`** с **системным промптом**, который говорит модели, что она может извлекать пользователей через инструменты.

---

### 14. Регистрация инструментов

Две обёртки **`defineChatSessionFunction`** вызывают **`fetchUserFromPrimary`** и **`fetchUserFromFallback`** с JSON Schema **`userId`**. **`functions`** передаётся в **`session.prompt`**, чтобы LLM мог вызывать инструменты по имени.

---

### 15. `promptLLM`

Оборачивает **`session.prompt`** с **`withTimeout`**, **`withRetries`** и ошибками с учётом correlation:

- Пустой обрезанный ответ > **`LLMCallError`** (повторяется).
- **`catch`**: **`classifyError`**; повторно бросает **`ToolExecutionError`** / **`LLMCallError`** без изменений; всё остальное становится **`LLMCallError`** с **`retryable`** только если нормализованный сбой был **`TIMEOUT`** (**`cause`** сохранён).
- **`retryOn`**: **`(err) => classifyError(err).retryable`**.

---

### 16. `runDegradedProfileResolution`

Запускается **без** LLM после того, как путь LLM упал с **`LLMCallError`**:

1. Извлекает **`u_<цифры>`** из совпадения **`SKIP_LLM_DEGRADED`** или свободного текста; иначе **`ValidationError`**.
2. **`withRetries`** + **`withTimeout`** для основного пути; **`retryOn`** только для **повторяющегося** **`ToolExecutionError`**.
3. Если основной всё ещё падает с повторяющейся ошибкой инструмента > пробует **`fetchUserFromFallback`**. Если запасной бросает > **`AgentWorkflowError`** (**`resolve_user_profile`**, **`cause`** = ошибка запасного).
4. Возвращает краткий ответ с префиксом **«Model unavailable; answered via deterministic fallback.»**.

---

### 17. `runAgent`

**Поток:**

1. **`correlationId = crypto.randomUUID()`**.
2. Пустой ввод > **`ValidationError`**.
3. Текст содержит **`u_demo_workflow`** > **`AgentWorkflowError`** (**`policy_guard`**) — демо-ограничитель после валидации.
4. **`SKIP_LLM_DEGRADED u_<цифры>`** > вынуждает **`LLMCallError`** без вызова модели (детерминированное деградированное демо).
5. Иначе **`promptLLM`**. Успех > **`{ ok: true, output }`**.
6. **`catch`** только **`LLMCallError`** > лог **`[degraded_mode]`**, вызов **`runDegradedProfileResolution`**, возврат **`ok: true`** с деградированным выводом.
7. Любая другая ошибка распространяется во внешний **`catch`**: **`classifyError`**, опциональный **`printAgentWorkflowErrorBanner`** для **`AgentWorkflowError`**, **`console.error("[agent_error]", …)`**, возврат **`{ ok: false, output: formatUserFacingError(...) }`**.

---

### 18. Демо-цикл и очистка

**`inputs`** запускает фиксированный набор строк (счастливый путь, **`u_999`**, **`u_demo_workflow`**, **`SKIP_LLM_DEGRADED u_777`**, пустой). Каждая итерация печатает **`USER:`**, **`runAgent`**, затем текст ассистента или текст ошибки.

**Очистка:** **`session`**, **`context`**, **`model`**, **`llama`** — важно для локальных/нативных привязок.

---

## Продемонстрированные ключевые концепции

### 1. Типизированные ошибки + стабильные коды

Дашборды и алерты могут группироваться по **`code`**. Пользователи никогда не видят **`details`** или стеки — только **`userMessage`** и **reference id**.

### 2. Классифицируйте, затем повторяйте

**`normalizeUnknownError` > `classifyError` > `retryable`** сохраняет согласованность **`withRetries`** и **`promptLLM`** в том, что считается временным.

### 3. Таймаут > повтор > запасной вариант > деградированный режим

**`withTimeout`** ограничивает время ожидания. **`withRetries`** обрабатывает нестабильные LLM или инструменты. **`runDegradedProfileResolution`** — это **детерминированный** путь, когда путь LLM непригоден, но Вы всё ещё можете завершить работу с инструментами.

### 4. `AgentWorkflowError` vs ошибки инструментов

**Инструмент** бросает **`ToolExecutionError`**. Когда **политика** блокирует или **основной + запасной** оба падают в оркестрированном деградированном потоке, выдаваемая ошибка — **`AgentWorkflowError`** с **`cause`**, указывающим на внутренний сбой.

---

## Ожидаемый вывод (типичный)

При запуске скрипта Вы увидите разделители, строки **`USER:`** и либо **`ASSISTANT:`**, либо **`ASSISTANT (error):`**. Для **`u_demo_workflow`** и для **`SKIP_LLM_DEGRADED u_777`** (когда запасной вариант падает) **stderr** показывает баннер **AGENT WORKFLOW FAILED** плюс JSON **`[agent_error]`**. Строки **`[retry]`** и **`[degraded_mode]`** появляются при активации повторов или деградированного пути.

Точная формулировка немного варьируется (например, вывод LLM на первом промпте зависит от модели).

---

## Лучшие практики

1. **Ограничивайте время** для вызовов LLM и инструментов (**`withTimeout`**).
2. **Повторяйте только временные сбои**; используйте **`classifyError`** (или эквивалент), чтобы ошибки валидации никогда не повторялись слепо.
3. **Jitter** для backoff, чтобы избежать синхронных повторов.
4. **Correlation ID** для каждой ошибки, видимой пользователю, и для структурированных логов.
5. **Разделяйте** логи оператора (**`[agent_error]`**, баннеры) от того, что Вы показываете конечным пользователям (**`formatUserFacingError`**).
6. **Освобождайте** нативные/модельные ресурсы при завершении скрипта.

---

## Почему это важно для AI-агентов

Агенты стакают **LLM + инструменты + оркестрация**. Сбои могут исходить из любого слоя; без таксономии и классификации Вы либо **повторяете всё** (расточительно), либо **ничего не повторяете** (хрупко). Этот пример показывает минимальный, но полный путь от **ошибок одиночного вызова** до **ошибки уровня рабочего процесса** **`AgentWorkflowError`** с ясным путём обновления до автоматов-предохранителей, реальной телеметрии и политика-типов уровня production.
