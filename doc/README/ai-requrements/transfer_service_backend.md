# Системные требования: Transfer Service (Микросервис переводов)

**Версия:** 1.0  
**Дата:** 24.04.2026  
**Связь с архитектурой:** [`c4_Level_2_containers_diagram_Card2CardTransfer_v1.plantuml`](c4_Level_2_containers_diagram_Card2CardTransfer_v1.plantuml)  
**Связь с компонентами:** [`c4_Level_3_component_diagram_Card2CardTransfer_(TransferService)_v1.plantuml`](c4_Level_3_component_diagram_Card2CardTransfer_(TransferService)_v1.plantuml)  
**Связь с BRD:** [`card2card_transfer_brd.md`](card2card_transfer_brd.md)  
**Связь с Use Cases:** [`card2card_transfer_uc.md`](card2card_transfer_uc.md)  

---

## 1. Общий обзор

### 1.1. Назначение
Transfer Service — ядро системы переводов с карты на карту. Сервис отвечает за:
- Прием и обработку запросов на переводы
- Валидацию входных данных (номер карты, сумма, лимиты)
- Маршрутизацию перевода (on-us / off-us / СБП)
- Конвертацию валют
- Координацию всех шагов процесса перевода через оркестратор
- Публикацию событий о переводах в Kafka

### 1.2. Высокоуровневая логика
```
Валидация → Проверка лимитов → Расчет комиссии → Проверка комплаенс → 
Маршрутизация → Выполнение перевода → Публикация события → Уведомление
```

### 1.3. Технологический стек
- **Язык:** Java 17+
- **Фреймворк:** Spring Boot 3.x
- **БД:** PostgreSQL (Transfer Database)
- **Очередь:** Kafka (публикация событий)
- **Интеграции:** REST (Feign Client / WebClient), Kafka Producer

---

## 2. Входные данные

### 2.1. REST API — Создание перевода

```typescript
POST /api/v1/transfers
Request Body:
{
  "sourceCardId": "UUID",           // ID карты отправителя (обязательно)
  "targetCardNumber": "string",     // Номер карты получателя 16-19 цифр (обязательно)
  "amount": "number",               // Сумма перевода > 0 (обязательно)
  "currency": "string",             // Код валюты ISO 4217 (обязательно)
  "comment": "string",              // Комментарий до 200 символов (опционально)
  "isMeToMe": "boolean",            // Признак перевода между своими картами (опционально)
  "saveAsTemplate": "boolean",      // Сохранить как шаблон (опционально)
  "confirmationCode": "string"      // Код подтверждения (обязательно для подтвержденных)
}
```

### 2.2. REST API — Получение статуса перевода

```typescript
GET /api/v1/transfers/{transferId}
Response:
{
  "transferId": "UUID",
  "status": "PENDING | PROCESSING | COMPLETED | FAILED | CANCELLED",
  "sourceCardId": "UUID",
  "targetCardMask": "string",       // Маскированный номер карты
  "amount": "number",
  "currency": "string",
  "fee": "number",
  "exchangeRate": "number",         // Курс конвертации (если применимо)
  "createdAt": "datetime",
  "completedAt": "datetime",
  "failureReason": "string"
}
```

### 2.3. REST API — Отмена перевода

```typescript
POST /api/v1/transfers/{transferId}/cancel
Request Body:
{
  "reason": "string"                // Причина отмены (опционально)
}
```

### 2.4. Внутренние параметры (из контекста)
- `userId` — UUID пользователя из JWT-токена
- `userRole` — роль пользователя (INDIVIDUAL / CORPORATE)
- `sessionId` — ID сессии для аудита

---

## 3. Валидации

### 3.1. Структурные валидации

| Поле | Проверка | Ошибка |
|------|----------|--------|
| `sourceCardId` | Должен быть валидным UUID | INVALID_CARD_ID |
| `targetCardNumber` | 16-19 цифр, только цифры | INVALID_CARD_NUMBER_FORMAT |
| `targetCardNumber` | Проходит алгоритм Луна | INVALID_CARD_NUMBER_CHECKSUM |
| `amount` | > 0 и <= 9999999.99 | INVALID_AMOUNT |
| `currency` | Существует в справочнике валют | UNSUPPORTED_CURRENCY |
| `comment` | <= 200 символов, без спецсимволов (<, >, &, ") | INVALID_COMMENT |

### 3.2. Бизнес-валидации

**3.2.1. Проверка карты отправителя:**
- Карта существует в системе банка
- Карта принадлежит текущему пользователю
- Статус карты = `ACTIVE` (не заблокирована, не просрочена)
- Срок действия карты не истек

**3.2.2. Проверка карты получателя:**
- Определение платежной системы (Visa / Mastercard / МИР) по BIN
- Определение банка-эквайера по BIN (on-us / off-us)
- Если on-us — проверка существования карты в системе банка
- Если off-us — проверка через платежный шлюз

**3.2.3. Проверка достаточности средств:**
```sql
SELECT balance, available_balance, currency 
FROM cards 
WHERE id = :sourceCardId AND user_id = :userId
```
- `available_balance >= amount + fee` (с учетом комиссии)

**3.2.4. Проверка me-to-me:**
- Если `isMeToMe = true` — обе карты должны принадлежать одному пользователю
- Карта-источник и карта-назначение — разные карты

### 3.3. Интеграционные валидации

**3.3.1. Проверка лимитов (через Limit Service):**
```http
GET /api/v1/limits/check?userId={userId}&amount={amount}&currency={currency}
```
Проверяемые лимиты:
- Лимит на один перевод
- Суточный лимит
- Месячный лимит
- Количество переводов в день

**3.3.2. Расчет комиссии (через Fee Service):**
```http
POST /api/v1/fees/calculate
{
  "sourceCardId": "UUID",
  "targetCardNumber": "string",
  "amount": "number",
  "currency": "string",
  "transferType": "ON_US | OFF_US | ME_TO_ME | SBP"
}
```

**3.3.3. Проверка комплаенс (через Compliance Service):**
```http
POST /api/v1/compliance/check-transfer
{
  "userId": "UUID",
  "sourceCardId": "UUID",
  "targetCardNumber": "string",
  "amount": "number",
  "currency": "string"
}
```

---

## 4. Основная логика

### 4.1. Процесс создания перевода (TransferOrchestrator)

```
ШАГ 1: ПРЕДВАЛИДАЦИЯ
  ├── Валидация формата номера карты (алгоритм Луна)
  ├── Валидация суммы и валюты
  └── Проверка, что карта отправителя существует и активна

ШАГ 2: ОПРЕДЕЛЕНИЕ ТИПА ПЕРЕВОДА (RoutingService)
  ├── Определение BIN карты получателя
  ├── Определение банка-эквайера
  ├── Если банк = наш → ON_US
  ├── Если банк ≠ наш → OFF_US
  ├── Если isMeToMe = true → ME_TO_ME
  └── Если сумма < лимита СБП → SBP (опционально)

ШАГ 3: ПРОВЕРКА ЛИМИТОВ (через Limit Service)
  ├── Запрос: GET /api/v1/limits/check
  ├── Если лимит превышен → FAILED (LIMIT_EXCEEDED)
  └── Если лимит в норме → резервирование лимита

ШАГ 4: РАСЧЕТ КОМИССИИ (через Fee Service)
  ├── Запрос: POST /api/v1/fees/calculate
  ├── Получение суммы комиссии
  └── Если me-to-me → комиссия = 0

ШАГ 5: ПРОВЕРКА ДОСТАТОЧНОСТИ СРЕДСТВ
  ├── Расчет: total = amount + fee
  ├── Если available_balance < total → FAILED (INSUFFICIENT_FUNDS)
  └── Если достаточно → резервирование средств

ШАГ 6: ПРОВЕРКА КОМПЛАЕНС (через Compliance Service)
  ├── Запрос: POST /api/v1/compliance/check-transfer
  ├── Если проверка не пройдена → FAILED (COMPLIANCE_REJECTED)
  └── Если пройдена → продолжение

ШАГ 7: СОЗДАНИЕ ТРАНЗАКЦИИ (TransferRepository)
  ├── Статус: PENDING
  ├── INSERT INTO transfers (...)
  └── Публикация события: TransferInitiated

ШАГ 8: ВЫПОЛНЕНИЕ ПЕРЕВОДА
  ├── ON_US: внутренняя проводка через Core Banking System
  │   ├── Списание с карты отправителя
  │   └── Зачисление на карту получателя
  ├── OFF_US: отправка через платежный шлюз
  │   ├── Формирование транзакции в формате ПС
  │   └── Отправка через PaymentGatewayClient
  └── SBP: отправка через СБП
      ├── Формирование запроса по API СБП
      └── Отправка через SBPClient

ШАГ 9: ОБРАБОТКА РЕЗУЛЬТАТА
  ├── Успех:
  │   ├── Статус: COMPLETED
  │   ├── UPDATE transfers SET status = 'COMPLETED'
  │   ├── Публикация события: TransferCompleted
  │   └── Отправка уведомлений (через Notification Service)
  └── Ошибка:
      ├── Статус: FAILED
      ├── UPDATE transfers SET status = 'FAILED', failure_reason = '...'
      ├── Отмена резервирования лимита
      ├── Отмена резервирования средств
      ├── Публикация события: TransferFailed
      └── Отправка уведомления об ошибке
```

### 4.2. Конвертация валют (CurrencyConverter)

```javascript
function convertCurrency(amount, sourceCurrency, targetCurrency) {
  // 1. Получение текущего курса из кэша (Redis, TTL: 1 минута)
  // 2. Если курс не в кэше — запрос к Core Banking System
  // 3. Расчет: convertedAmount = amount * exchangeRate
  // 4. Округление до 2 знаков (банковское округление)
  // 5. Возврат: { convertedAmount, exchangeRate, fee }
}
```

### 4.3. Маршрутизация (RoutingService)

```javascript
function determineRoute(sourceCard, targetCardBIN, amount, currency) {
  // 1. Определение банка по BIN
  const bankInfo = binService.lookup(targetCardBIN);
  
  // 2. Если банк = наш банк → ON_US
  if (bankInfo.bankId === OUR_BANK_ID) {
    return { type: 'ON_US', bankName: bankInfo.bankName };
  }
  
  // 3. Если сумма < лимита СБП и СБП доступна → SBP
  if (amount <= SBP_LIMIT && sbpAvailable) {
    return { type: 'SBP', bankName: bankInfo.bankName };
  }
  
  // 4. Иначе → OFF_US через платежную систему
  const paymentSystem = detectPaymentSystem(targetCardBIN);
  return { type: 'OFF_US', paymentSystem, bankName: bankInfo.bankName };
}
```

---

## 5. Интеграции

### 5.1. Внутренние сервисы

| Сервис | Протокол | Метод | Назначение |
|--------|----------|-------|------------|
| **Limit Service** | REST (Feign Client) | `GET /api/v1/limits/check` | Проверка лимитов |
| **Fee Service** | REST (Feign Client) | `POST /api/v1/fees/calculate` | Расчет комиссии |
| **Compliance Service** | REST (Feign Client) | `POST /api/v1/compliance/check-transfer` | Проверка комплаенс |
| **Notification Service** | Kafka | Публикация события | Отправка уведомлений |

### 5.2. Внешние системы

| Система | Протокол | Назначение |
|---------|----------|------------|
| **Платежный шлюз (Visa/MC/МИР)** | REST API | Отправка off-us транзакций |
| **СБП** | API СБП | Отправка переводов через СБП |
| **Core Banking System (АБС)** | API | Внутренние проводки, проверка баланса |

### 5.3. Kafka события

**Топик:** `banking.transfer.events`

```yaml
Топик: banking.transfer.events
Партиции: 12 (по transferId hash)
Replication Factor: 3
Retention: 30 дней
```

**Типы событий:**

| Тип события | Payload | Назначение |
|-------------|---------|------------|
| `TransferInitiated` | transferId, sourceCardId, targetCardMask, amount, currency, fee | Начало обработки |
| `TransferCompleted` | transferId, status, completedAt, routingType | Успешное завершение |
| `TransferFailed` | transferId, failureReason, failedAt | Ошибка обработки |
| `TransferCancelled` | transferId, reason, cancelledAt | Отмена перевода |

**Пример события:**
```json
{
  "eventType": "TransferCompleted",
  "source": "transfer-service",
  "timestamp": "2026-04-24T10:30:00Z",
  "payload": {
    "transferId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "status": "COMPLETED",
    "amount": 1500.00,
    "currency": "RUB",
    "fee": 0.00,
    "routingType": "ON_US",
    "sourceCardMask": "****1234",
    "targetCardMask": "****5678",
    "completedAt": "2026-04-24T10:30:05Z"
  }
}
```

---

## 6. Исключительные ситуации

### 6.1. Валидационные ошибки (HTTP 400)

| Код ошибки | Сообщение | Условие |
|------------|-----------|---------|
| `INVALID_CARD_NUMBER` | "Неверный номер карты. Проверьте введенные данные" | Ошибка алгоритма Луна |
| `INVALID_AMOUNT` | "Сумма перевода должна быть больше 0" | amount <= 0 |
| `UNSUPPORTED_CURRENCY` | "Валюта не поддерживается" | Валюта не в справочнике |
| `INVALID_COMMENT` | "Комментарий содержит недопустимые символы" | Спецсимволы в комментарии |

### 6.2. Бизнес-ошибки (HTTP 422)

| Код ошибки | Сообщение | Условие |
|------------|-----------|---------|
| `CARD_NOT_FOUND` | "Карта отправителя не найдена" | Карта не существует |
| `CARD_NOT_ACTIVE` | "Карта отправителя заблокирована или просрочена" | Статус карты ≠ ACTIVE |
| `INSUFFICIENT_FUNDS` | "Недостаточно средств на карте с учетом комиссии" | balance < amount + fee |
| `LIMIT_EXCEEDED` | "Превышен лимит. Максимальная сумма: X" | Превышение суточного/месячного лимита |
| `COMPLIANCE_REJECTED` | "Операция отклонена службой безопасности" | Отказ комплаенс-проверки |
| `CARD_NOT_ACTIVE` | "Карта получателя недоступна для зачисления" | Карта получателя заблокирована |

### 6.3. Интеграционные ошибки (HTTP 502/503)

| Код ошибки | Сообщение | Условие |
|------------|-----------|---------|
| `LIMIT_SERVICE_UNAVAILABLE` | "Сервис проверки лимитов временно недоступен" | Limit Service не отвечает |
| `FEE_SERVICE_UNAVAILABLE` | "Сервис расчета комиссии временно недоступен" | Fee Service не отвечает |
| `COMPLIANCE_SERVICE_UNAVAILABLE` | "Сервис проверки безопасности временно недоступен" | Compliance Service не отвечает |
| `PAYMENT_GATEWAY_UNAVAILABLE` | "Платежный шлюз временно недоступен. Попробуйте позже" | Платежный шлюз не отвечает |
| `CORE_BANKING_UNAVAILABLE` | "Банковская система временно недоступна" | АБС не отвечает |

### 6.4. Стратегии восстановления

| Тип ошибки | Стратегия | Действие |
|------------|-----------|----------|
| **Таймаут интеграции** | Retry (3 попытки) | Exponential backoff: 1s, 2s, 4s |
| **Недоступность сервиса** | Circuit Breaker | Переключение на fallback (отложенная обработка) |
| **Ошибка БД** | Transaction rollback | Откат всех изменений, повторная попытка |
| **Конфликт данных** | Optimistic locking | Повторная попытка с обновленными данными |
| **Недоступность Kafka** | Dead Letter Queue | Сохранение события в DLQ для повторной отправки |

---

## 7. Выходные данные

### 7.1. Успешный ответ (HTTP 201)

```json
{
  "transferId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "status": "COMPLETED",
  "amount": 1500.00,
  "currency": "RUB",
  "fee": 0.00,
  "exchangeRate": null,
  "sourceCardMask": "****1234",
  "targetCardMask": "****5678",
  "routingType": "ON_US",
  "createdAt": "2026-04-24T10:30:00Z",
  "completedAt": "2026-04-24T10:30:05Z"
}
```

### 7.2. Ошибка валидации (HTTP 400)

```json
{
  "error": "VALIDATION_ERROR",
  "code": "INVALID_CARD_NUMBER",
  "message": "Неверный номер карты. Проверьте введенные данные",
  "details": {
    "field": "targetCardNumber",
    "value": "4111111111111112",
    "reason": "Не прошел проверку алгоритмом Луна"
  }
}
```

### 7.3. Бизнес-ошибка (HTTP 422)

```json
{
  "error": "BUSINESS_ERROR",
  "code": "INSUFFICIENT_FUNDS",
  "message": "Недостаточно средств на карте с учетом комиссии",
  "details": {
    "availableBalance": 1000.00,
    "requiredAmount": 1500.00,
    "fee": 15.00,
    "totalRequired": 1515.00
  }
}
```

### 7.4. Kafka события (публикуемые)

| Событие | Топик | Формат |
|---------|-------|--------|
| TransferInitiated | banking.transfer.events | JSON с transferId, sourceCardId, amount |
| TransferCompleted | banking.transfer.events | JSON с transferId, status, completedAt |
| TransferFailed | banking.transfer.events | JSON с transferId, failureReason |
| TransferCancelled | banking.transfer.events | JSON с transferId, reason |

---

## 8. Производительность

### 8.1. Целевые показатели

| Метрика | Целевое значение | Метод измерения |
|---------|-----------------|-----------------|
| Время обработки on-us перевода | ≤ 5 секунд (p99) | Мониторинг времени транзакции |
| Время обработки off-us перевода | ≤ 30 секунд (p99) | Мониторинг времени транзакции |
| Пропускная способность | ≥ 1000 транзакций/сек | Нагрузочное тестирование |
| Время отклика API | ≤ 200 мс (p95) | APM мониторинг |
| Доступность сервиса | 99.9% | Health check мониторинг |

### 8.2. Оптимизации

- **Кэширование BIN-справочника** в памяти сервиса (обновление раз в час)
- **Кэширование курсов валют** (Redis, TTL: 1 минута)
- **Пул соединений к БД** (HikariCP, max: 20 соединений)
- **Batch-обработка** при записи в историю
- **Асинхронная отправка уведомлений** через Kafka
- **Индексы БД**: (user_id, created_at), (status), (source_card_id)

### 8.3. Ограничения

| Параметр | Значение |
|----------|----------|
| Максимальный размер запроса | 10 KB |
| Максимальное время обработки | 60 секунд |
| Максимальное количество одновременных запросов | 500 |
| Таймаут интеграции с внешними системами | 10 секунд |
| Размер пула потоков | 50 потоков |

---

## 9. Безопасность

### 9.1. Аутентификация и авторизация
- Все endpoints требуют JWT-токен (проверка через API Gateway)
- Проверка принадлежности карты пользователю (userId из токена)
- Rate limiting: 10 запросов в минуту на пользователя

### 9.2. Защита данных
- PAN-номера карт не логируются (только маскированные: ****1234)
- Все sensitive данные передаются только по TLS 1.3
- PCI DSS compliance: хранение PAN только в HSM

### 9.3. Аудит
- Логирование всех операций с transferId, userId, timestamp
- Аудит изменений статуса перевода
- Сохранение failureReason для анализа ошибок