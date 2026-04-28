# Системные требования: Limit Service (Микросервис управления лимитами)

**Версия:** 1.0  
**Дата:** 28.04.2026  
**Связь с архитектурой:** [`c4_Level_2_containers_diagram_Card2CardTransfer_v1.plantuml`](c4_Level_2_containers_diagram_Card2CardTransfer_v1.plantuml)  
**Связь с BRD:** [`card2card_transfer_brd.md`](card2card_transfer_brd.md)  
**Связь с Transfer Service:** [`transfer_service_backend.md`](transfer_service_backend.md)  
**Связь с Fee Service:** [`fee_service_backend.md`](fee_service_backend.md)  

---

## 1. Общий обзор

### 1.1. Назначение
Limit Service — микросервис для управления и проверки лимитов переводов с карты на карту. Сервис отвечает за:
- Проверку лимитов перед выполнением перевода (на один перевод, суточные, месячные)
- Резервирование и списание лимитов при выполнении перевода
- Возврат лимитов при отмене/ошибке перевода
- Управление лимитами через административную панель
- Хранение истории использования лимитов

### 1.2. Высокоуровневая логика
```
Проверка лимита → Резервирование лимита → Фиксация использования → 
Возврат лимита (при ошибке/отмене)
```

### 1.3. Технологический стек
- **Язык:** Java 17+
- **Фреймворк:** Spring Boot 3.x
- **БД:** PostgreSQL (Limit Database)
- **Очередь:** Kafka Consumer (подписка на события переводов)
- **Интеграции:** REST (Feign Client), Kafka Consumer

---

## 2. Входные данные

### 2.1. REST API — Проверка лимита

```typescript
GET /api/v1/limits/check?userId={UUID}&amount={number}&currency={string}&transferType={string}
Response:
{
  "allowed": true | false,
  "limitType": "SINGLE | DAILY | MONTHLY",
  "limitValue": 150000.00,
  "usedValue": 50000.00,
  "remainingValue": 100000.00,
  "message": "Лимит не превышен"
}
```

### 2.2. REST API — Резервирование лимита

```typescript
POST /api/v1/limits/reserve
Request Body:
{
  "userId": "UUID",
  "transferId": "UUID",
  "amount": 1500.00,
  "currency": "RUB",
  "transferType": "OFF_US"
}
Response:
{
  "reserved": true,
  "reservationId": "UUID",
  "remainingDailyLimit": 98500.00,
  "remainingMonthlyLimit": 498500.00
}
```

### 2.3. REST API — Возврат лимита (при ошибке/отмене)

```typescript
POST /api/v1/limits/release
Request Body:
{
  "transferId": "UUID",
  "reason": "TRANSFER_FAILED | TRANSFER_CANCELLED"
}
Response:
{
  "released": true,
  "releasedAmount": 1500.00
}
```

### 2.4. REST API — Управление лимитами (административные)

```typescript
// Получение лимитов пользователя
GET /api/v1/admin/limits/{userId}

// Установка лимитов пользователя
PUT /api/v1/admin/limits/{userId}
Request Body:
{
  "singleTransferLimit": 150000.00,
  "dailyLimit": 500000.00,
  "monthlyLimit": 5000000.00,
  "dailyCountLimit": 50,
  "currencyLimits": {
    "USD": { "dailyLimit": 10000.00, "monthlyLimit": 100000.00 },
    "EUR": { "dailyLimit": 10000.00, "monthlyLimit": 100000.00 }
  }
}

// Получение глобальных лимитов системы
GET /api/v1/admin/limits/global

// Установка глобальных лимитов
PUT /api/v1/admin/limits/global
```

### 2.5. Kafka Consumer — События переводов

```yaml
Топик: banking.transfer.events
Группа: limit-service-group
Типы событий:
  - TransferInitiated  # Резервирование лимита
  - TransferCompleted  # Фиксация использования лимита
  - TransferFailed     # Возврат лимита
  - TransferCancelled  # Возврат лимита
```

---

## 3. Валидации

### 3.1. Структурные валидации

| Поле | Проверка | Ошибка |
|------|----------|--------|
| `userId` | Валидный UUID | INVALID_USER_ID |
| `transferId` | Валидный UUID | INVALID_TRANSFER_ID |
| `amount` | > 0 и <= 9999999.99 | INVALID_AMOUNT |
| `currency` | Существует в справочнике | UNSUPPORTED_CURRENCY |
| `transferType` | Одно из: ON_US, OFF_US, ME_TO_ME, SBP | INVALID_TRANSFER_TYPE |
| `limitValue` | > 0 | INVALID_LIMIT_VALUE |

### 3.2. Бизнес-валидации

**3.2.1. Проверка лимита на один перевод:**
```sql
SELECT single_transfer_limit 
FROM user_limits 
WHERE user_id = :userId
```
- Если `amount > singleTransferLimit` → лимит превышен

**3.2.2. Проверка суточного лимита:**
```sql
SELECT COALESCE(SUM(amount), 0) as daily_total
FROM limit_usage 
WHERE user_id = :userId 
  AND created_at >= CURRENT_DATE
  AND currency = :currency
```
- Если `daily_total + amount > dailyLimit` → лимит превышен

**3.2.3. Проверка месячного лимита:**
```sql
SELECT COALESCE(SUM(amount), 0) as monthly_total
FROM limit_usage 
WHERE user_id = :userId 
  AND created_at >= DATE_TRUNC('month', CURRENT_DATE)
  AND currency = :currency
```
- Если `monthly_total + amount > monthlyLimit` → лимит превышен

**3.2.4. Проверка количества переводов в день:**
```sql
SELECT COUNT(*) as daily_count
FROM limit_usage 
WHERE user_id = :userId 
  AND created_at >= CURRENT_DATE
```
- Если `daily_count + 1 > dailyCountLimit` → лимит превышен

**3.2.5. Валютные лимиты:**
- Для USD и EUR — отдельные лимиты (валютный контроль)
- Если сумма в USD > 10000 — требуется дополнительная проверка Compliance Service

### 3.3. Административные валидации

**3.3.1. Валидация значений лимитов:**
- `singleTransferLimit <= dailyLimit` (логическая проверка)
- `dailyLimit <= monthlyLimit`
- Все значения > 0
- `dailyCountLimit` > 0 и <= 1000

**3.3.2. Конфликт лимитов:**
- Персональные лимиты пользователя имеют приоритет над глобальными
- Если персональный лимит не установлен — применяется глобальный

---

## 4. Основная логика

### 4.1. Процесс проверки лимита (LimitCheckService)

```
ШАГ 1: ПОЛУЧЕНИЕ ЛИМИТОВ ПОЛЬЗОВАТЕЛЯ
  ├── Поиск персональных лимитов пользователя
  ├── Если не найдены → загрузка глобальных лимитов
  └── Если и глобальные не найдены → DEFAULT лимиты

ШАГ 2: ПРОВЕРКА ЛИМИТА НА ОДИН ПЕРЕВОД
  ├── Если amount > singleTransferLimit → BLOCKED
  └── Если amount <= singleTransferLimit → OK

ШАГ 3: ПРОВЕРКА СУТОЧНОГО ЛИМИТА
  ├── Расчет: dailyUsed = SUM(amount) за сегодня
  ├── Если dailyUsed + amount > dailyLimit → BLOCKED
  └── Если dailyUsed + amount <= dailyLimit → OK

ШАГ 4: ПРОВЕРКА МЕСЯЧНОГО ЛИМИТА
  ├── Расчет: monthlyUsed = SUM(amount) за месяц
  ├── Если monthlyUsed + amount > monthlyLimit → BLOCKED
  └── Если monthlyUsed + amount <= monthlyLimit → OK

ШАГ 5: ПРОВЕРКА КОЛИЧЕСТВА ПЕРЕВОДОВ
  ├── Расчет: dailyCount = COUNT(transfers) за сегодня
  ├── Если dailyCount + 1 > dailyCountLimit → BLOCKED
  └── Если dailyCount + 1 <= dailyCountLimit → OK

ШАГ 6: ПРОВЕРКА ВАЛЮТНЫХ ЛИМИТОВ
  ├── Если currency = USD или EUR:
  │   ├── Проверка отдельного лимита для валюты
  │   └── Если amount > currencyLimit → BLOCKED
  └── Иначе → OK

ШАГ 7: ВОЗВРАТ РЕЗУЛЬТАТА
  └── { allowed: true/false, limitType, limitValue, usedValue, remainingValue }
```

### 4.2. Процесс резервирования лимита (LimitReservationService)

```
ШАГ 1: ПРОВЕРКА ДУБЛИРОВАНИЯ
  ├── Поиск существующей резервации по transferId
  ├── Если найдена → возврат существующей резервации
  └── Если не найдена → создание новой

ШАГ 2: СОЗДАНИЕ ЗАПИСИ РЕЗЕРВАЦИИ
  ├── INSERT INTO limit_reservations (transferId, userId, amount, currency, status)
  └── Статус: RESERVED

ШАГ 3: ВОЗВРАТ РЕЗУЛЬТАТА
  └── { reserved: true, reservationId, remainingDailyLimit, remainingMonthlyLimit }
```

### 4.3. Процесс фиксации использования лимита (LimitUsageService)

```
ШАГ 1: ПОЛУЧЕНИЕ СОБЫТИЯ ИЗ KAFKA (TransferCompleted)
  ├── Поиск резервации по transferId
  ├── Если не найдена → создание записи использования без резервации
  └── Если найдена → переход к шагу 2

ШАГ 2: ФИКСАЦИЯ ИСПОЛЬЗОВАНИЯ
  ├── INSERT INTO limit_usage (userId, transferId, amount, currency, transferType)
  ├── UPDATE limit_reservations SET status = 'USED'
  └── Обновление кэша лимитов

ШАГ 3: ПРОВЕРКА ПОРОГОВЫХ ЗНАЧЕНИЙ
  ├── Если usage > 80% лимита → публикация события LimitWarning
  ├── Если usage > 95% лимита → публикация события LimitCritical
  └── Если usage = 100% → публикация события LimitExhausted
```

### 4.4. Процесс возврата лимита (LimitReleaseService)

```
ШАГ 1: ПОЛУЧЕНИЕ СОБЫТИЯ (TransferFailed / TransferCancelled)
  ├── Поиск резервации по transferId
  ├── Если не найдена → ошибка (резервация не существует)
  └── Если найдена → переход к шагу 2

ШАГ 2: ОСВОБОЖДЕНИЕ ЛИМИТА
  ├── UPDATE limit_reservations SET status = 'RELEASED'
  └── Сумма лимита возвращается в доступный остаток

ШАГ 3: ЛОГИРОВАНИЕ
  └── INSERT INTO limit_audit (transferId, userId, amount, action, reason)
```

---

## 5. Интеграции

### 5.1. Внутренние сервисы

| Сервис | Протокол | Метод | Назначение |
|--------|----------|-------|------------|
| **Transfer Service** | REST | `GET /api/v1/limits/check` | Проверка лимитов перед переводом |
| **Transfer Service** | REST | `POST /api/v1/limits/reserve` | Резервирование лимита |
| **Transfer Service** | REST | `POST /api/v1/limits/release` | Возврат лимита |
| **Fee Service** | REST | `GET /api/v1/limits/check` | Проверка лимитов для комиссии |
| **Admin Panel** | REST | `GET/PUT /api/v1/admin/limits/*` | Управление лимитами |

### 5.2. Kafka

**Топик:** `banking.transfer.events` (Consumer)

```yaml
Группа: limit-service-group
Client ID: limit-service
Auto Commit: false
Max Poll Records: 100
```

**Обработка событий:**

| Событие | Действие |
|---------|----------|
| `TransferInitiated` | Резервирование лимита (если не вызвано через REST) |
| `TransferCompleted` | Фиксация использования лимита |
| `TransferFailed` | Возврат лимита |
| `TransferCancelled` | Возврат лимита |

### 5.3. База данных

```sql
-- Таблица лимитов пользователей
CREATE TABLE user_limits (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL UNIQUE,
    single_transfer_limit DECIMAL(10,2) NOT NULL DEFAULT 150000.00,
    daily_limit DECIMAL(10,2) NOT NULL DEFAULT 500000.00,
    monthly_limit DECIMAL(10,2) NOT NULL DEFAULT 5000000.00,
    daily_count_limit INTEGER NOT NULL DEFAULT 50,
    currency_limits JSONB,              -- { "USD": { "daily": 10000, "monthly": 100000 } }
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица глобальных лимитов системы
CREATE TABLE global_limits (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    single_transfer_limit DECIMAL(10,2) NOT NULL DEFAULT 150000.00,
    daily_limit DECIMAL(10,2) NOT NULL DEFAULT 500000.00,
    monthly_limit DECIMAL(10,2) NOT NULL DEFAULT 5000000.00,
    daily_count_limit INTEGER NOT NULL DEFAULT 50,
    currency_limits JSONB,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица резерваций лимитов
CREATE TABLE limit_reservations (
    id UUID PRIMARY KEY,
    transfer_id UUID NOT NULL UNIQUE,
    user_id UUID NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'RESERVED',  -- RESERVED, USED, RELEASED, EXPIRED
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP DEFAULT (CURRENT_TIMESTAMP + INTERVAL '1 hour')
);

-- Таблица использования лимитов
CREATE TABLE limit_usage (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    transfer_id UUID NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    transfer_type VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица аудита лимитов
CREATE TABLE limit_audit (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    transfer_id UUID,
    amount DECIMAL(10,2),
    action VARCHAR(50) NOT NULL,         -- CHECK, RESERVE, USE, RELEASE, EXPIRE
    reason VARCHAR(255),
    details JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Индексы
CREATE INDEX idx_user_limits_user_id ON user_limits(user_id);
CREATE INDEX idx_limit_reservations_transfer_id ON limit_reservations(transfer_id);
CREATE INDEX idx_limit_reservations_status ON limit_reservations(status);
CREATE INDEX idx_limit_usage_user_id_date ON limit_usage(user_id, created_at);
CREATE INDEX idx_limit_usage_user_id_currency ON limit_usage(user_id, currency, created_at);
CREATE INDEX idx_limit_audit_user_id ON limit_audit(user_id);
CREATE INDEX idx_limit_audit_action ON limit_audit(action);
```

---

## 6. Исключительные ситуации

### 6.1. Валидационные ошибки (HTTP 400)

| Код ошибки | Сообщение | Условие |
|------------|-----------|---------|
| `INVALID_USER_ID` | "Некорректный идентификатор пользователя" | userId не UUID |
| `INVALID_TRANSFER_ID` | "Некорректный идентификатор перевода" | transferId не UUID |
| `INVALID_AMOUNT` | "Некорректная сумма перевода" | amount <= 0 или > 9999999.99 |
| `UNSUPPORTED_CURRENCY` | "Валюта не поддерживается" | currency отсутствует в справочнике |
| `INVALID_TRANSFER_TYPE` | "Некорректный тип перевода" | transferType не из допустимого списка |
| `INVALID_LIMIT_VALUE` | "Некорректное значение лимита" | limitValue <= 0 |

### 6.2. Бизнес-ошибки (HTTP 422)

| Код ошибки | Сообщение | Условие |
|------------|-----------|---------|
| `LIMIT_EXCEEDED_SINGLE` | "Превышен лимит на один перевод" | amount > singleTransferLimit |
| `LIMIT_EXCEEDED_DAILY` | "Превышен суточный лимит переводов" | dailyUsed + amount > dailyLimit |
| `LIMIT_EXCEEDED_MONTHLY` | "Превышен месячный лимит переводов" | monthlyUsed + amount > monthlyLimit |
| `LIMIT_EXCEEDED_COUNT` | "Превышено количество переводов в день" | dailyCount + 1 > dailyCountLimit |
| `LIMIT_EXCEEDED_CURRENCY` | "Превышен лимит для валюты" | amount > currencyLimit |
| `RESERVATION_NOT_FOUND` | "Резервация не найдена" | transferId отсутствует в limit_reservations |
| `RESERVATION_ALREADY_USED` | "Резервация уже использована" | status = 'USED' |
| `RESERVATION_ALREADY_RELEASED` | "Резервация уже освобождена" | status = 'RELEASED' |
| `RESERVATION_EXPIRED` | "Срок резервации истёк" | expires_at < NOW() |
| `LIMIT_CONFLICT` | "Конфликт лимитов: singleTransferLimit > dailyLimit" | Логическая ошибка в настройках |

### 6.3. Интеграционные ошибки (HTTP 502/503)

| Код ошибки | Сообщение | Условие | Действие |
|------------|-----------|---------|----------|
| `DATABASE_TIMEOUT` | "Таймаут при запросе к БД" | PostgreSQL не отвечает > 2 сек | Retry 3 раза, затем Circuit Breaker |
| `DATABASE_CONNECTION` | "Ошибка подключения к БД" | Connection pool исчерпан | Retry с exponential backoff |
| `KAFKA_CONSUMER_ERROR` | "Ошибка обработки события Kafka" | Ошибка десериализации | Отправка в DLQ |
| `KAFKA_TIMEOUT` | "Таймаут при обработке Kafka" | Consumer не отвечает > 5 сек | Перезапуск consumer |
| `CACHE_UNAVAILABLE` | "Кэш недоступен" | Redis не отвечает | Graceful degradation (чтение из БД) |

### 6.4. Стратегии восстановления

**6.4.1. Транзакционные откаты:**
- При ошибке в процессе резервирования → автоматический ROLLBACK
- При ошибке фиксации использования → повторная попытка через Kafka retry

**6.4.2. Компенсирующие операции:**
- При ошибке перевода → автоматический возврат лимита (release)
- При истечении резервации (1 час) → автоматический release через scheduler

**6.4.3. Повторные попытки:**
- REST запросы: до 3 раз с exponential backoff (100ms, 500ms, 2s)
- Kafka consumer: до 5 раз с backoff (1s, 2s, 4s, 8s, 16s), затем DLQ

**6.4.4. Graceful degradation:**
- При недоступности БД → возврат кэшированных лимитов (TTL: 5 минут)
- При недоступности Kafka → синхронная обработка через REST
- При превышении времени ответа → возврат BLOCKED (безопасная сторона)

---

## 7. Выходные данные

### 7.1. REST API — Успешные ответы

**7.1.1. Проверка лимита (200):**
```json
{
  "allowed": true,
  "limitType": "DAILY",
  "limitValue": 500000.00,
  "usedValue": 150000.00,
  "remainingValue": 350000.00,
  "message": "Лимит не превышен",
  "checkedAt": "2026-04-28T14:30:00Z"
}
```

**7.1.2. Резервирование лимита (201):**
```json
{
  "reserved": true,
  "reservationId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "remainingDailyLimit": 98500.00,
  "remainingMonthlyLimit": 498500.00,
  "reservedAt": "2026-04-28T14:30:00Z"
}
```

**7.1.3. Возврат лимита (200):**
```json
{
  "released": true,
  "releasedAmount": 1500.00,
  "releasedAt": "2026-04-28T14:35:00Z"
}
```

### 7.2. REST API — Ошибки

**7.2.1. Лимит превышен (422):**
```json
{
  "error": "LIMIT_EXCEEDED_DAILY",
  "message": "Превышен суточный лимит переводов",
  "details": {
    "limitType": "DAILY",
    "limitValue": 500000.00,
    "usedValue": 485000.00,
    "requestedAmount": 50000.00,
    "remainingValue": 15000.00
  },
  "timestamp": "2026-04-28T14:30:00Z"
}
```

**7.2.2. Ошибка валидации (400):**
```json
{
  "error": "INVALID_AMOUNT",
  "message": "Некорректная сумма перевода",
  "details": {
    "field": "amount",
    "value": -100.00,
    "constraints": {
      "min": 0.01,
      "max": 9999999.99
    }
  },
  "timestamp": "2026-04-28T14:30:00Z"
}
```

### 7.3. Kafka события (Producer)

**Топик:** `banking.limit.events`

| Событие | Условие | Payload |
|---------|---------|---------|
| `LimitWarning` | Использовано > 80% лимита | `{ userId, limitType, usedPercent, remainingValue }` |
| `LimitCritical` | Использовано > 95% лимита | `{ userId, limitType, usedPercent, remainingValue }` |
| `LimitExhausted` | Лимит исчерпан (100%) | `{ userId, limitType, limitValue, usedValue }` |
| `LimitReset` | Лимит сброшен (новый день/месяц) | `{ userId, limitType, resetValue }` |

---

## 8. Производительность

### 8.1. Целевые показатели

| Метрика | Целевое значение | Метод измерения |
|---------|-----------------|-----------------|
| Время проверки лимита | ≤ 50 мс (p99) | JMeter, Gatling |
| Время резервирования | ≤ 100 мс (p99) | JMeter, Gatling |
| Время возврата лимита | ≤ 100 мс (p99) | JMeter, Gatling |
| Пропускная способность | 3000 TPS | Load testing |
| Время ответа при пике | ≤ 200 мс (p99) | 2x нормальная нагрузка |
| Доступность | 99.95% | Uptime monitoring |

### 8.2. Оптимизации

**8.2.1. Кэширование:**
- **Redis Cache:** лимиты пользователя (TTL: 5 минут)
- **Local Cache (Caffeine):** глобальные лимиты (TTL: 1 минута)
- **Кэш использования:** агрегированные суммы за день/месяц (TTL: 1 минута)

**8.2.2. Индексы БД:**
- `(user_id, created_at)` — для быстрых агрегаций по времени
- `(user_id, currency, created_at)` — для валютных лимитов
- `(transfer_id)` — для поиска резерваций
- `(status)` — для фоновых задач по очистке

**8.2.3. Пул соединений:**
- HikariCP: максимум 20 соединений
- Таймаут: 2 секунды
- Idle timeout: 10 минут

**8.2.4. Пакетная обработка:**
- Kafka consumer: max.poll.records = 100
- Batch insert для limit_usage: до 50 записей за транзакцию

### 8.3. Ограничения

| Параметр | Значение | Обоснование |
|----------|----------|-------------|
| Максимальная нагрузка | 5000 TPS | Исходя из пиковых нагрузок (Черная пятница) |
| Максимальное время ожидания | 500 мс | Таймаут для Transfer Service |
| Максимальный размер запроса | 10 KB | Ограничение на body запроса |
| Максимальное количество резерваций | 1000 на пользователя | Защита от утечки ресурсов |
| TTL резервации | 1 час | Автоматическая очистка "зависших" резерваций |

---

## 9. Безопасность

### 9.1. Аутентификация и авторизация

| Endpoint | Метод | Роль | Аутентификация |
|----------|-------|------|----------------|
| `GET /api/v1/limits/check` | Внутренний | Transfer Service | Service-to-Service JWT |
| `POST /api/v1/limits/reserve` | Внутренний | Transfer Service | Service-to-Service JWT |
| `POST /api/v1/limits/release` | Внутренний | Transfer Service | Service-to-Service JWT |
| `GET /api/v1/admin/limits/*` | Административный | ADMIN | JWT + Role check |
| `PUT /api/v1/admin/limits/*` | Административный | ADMIN | JWT + Role check |

### 9.2. Аудит

Все операции с лимитами логируются в таблицу `limit_audit`:
- Изменение лимитов (кто, когда, какие значения)
- Проверки лимитов (только превышения)
- Резервирования и возвраты
- Истечение резерваций (scheduler)

### 9.3. Rate Limiting

| Endpoint | Лимит | Период |
|----------|-------|--------|
| `GET /api/v1/limits/check` | 5000 запросов | 1 минута |
| `POST /api/v1/limits/reserve` | 1000 запросов | 1 минута |
| `POST /api/v1/limits/release` | 1000 запросов | 1 минута |
| `GET /api/v1/admin/limits/*` | 100 запросов | 1 минута |
| `PUT /api/v1/admin/limits/*` | 50 запросов | 1 минута |

### 9.4. Защита данных

- **PCI DSS:** лимиты не содержат PCI-чувствительных данных
- **Шифрование:** данные в покое зашифрованы (AES-256)
- **Маскирование:** суммы лимитов доступны только авторизованным сервисам
- **Логирование:** без записи персональных данных в логи