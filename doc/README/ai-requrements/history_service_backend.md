# Системные требования: History Service (Микросервис истории переводов)

**Версия:** 1.2  
**Дата:** 30.04.2026  
**Связь с архитектурой:** [`c4_Level_2_containers_diagram_Card2CardTransfer_v1.plantuml`](c4_Level_2_containers_diagram_Card2CardTransfer_v1.plantuml)  
**Связь с BRD:** [`card2card_transfer_brd.md`](card2card_transfer_brd.md)  
**Связь с Transfer Service:** [`transfer_service_backend.md`](transfer_service_backend.md)  

---

## 1. Общий обзор

### 1.1. Назначение
History Service — микросервис для хранения и предоставления истории переводов с карты на карту. Сервис отвечает за:
- Хранение полной истории всех переводов пользователя
- Фильтрацию, поиск и сортировку истории
- Экспорт истории в PDF и CSV форматы
- Кэширование часто запрашиваемых данных
- Обеспечение быстрого доступа к истории (пагинация, индексы)

### 1.2. Высокоуровневая логика
```
Получение события → Сохранение в БД → Индексация → 
Обновление кэша → Готовность к запросам
```

### 1.3. Технологический стек
- **Язык:** Java 17+
- **Фреймворк:** Spring Boot 3.x
- **БД:** PostgreSQL (History Database)
- **Очередь:** Kafka Consumer (подписка на события переводов)
- **Кэш:** Redis (последние 50 переводов пользователя)
- **Генерация отчётов:** Apache POI (Excel), iText (PDF)

---

## 2. Входные данные

### 2.1. Kafka Consumer — События переводов

```yaml
Топик: banking.transfer.events
Группа: history-service-group
Типы событий:
  - TransferInitiated       # Создание записи со статусом PENDING
  - TransferCompleted       # Обновление статуса на COMPLETED
  - TransferFailed          # Обновление статуса на FAILED
  - TransferCancelled       # Обновление статуса на CANCELLED
```

### 2.2. REST API — Получение истории

```typescript
// Получение истории переводов пользователя
GET /api/v1/history?userId={UUID}&page=1&size=20&from=2026-01-01&to=2026-04-30
  &status=COMPLETED&currency=RUB&sortBy=createdAt&sortDir=DESC

Response:
{
  "content": [
    {
      "transferId": "UUID",
      "fromCard": "****1234",
      "toCard": "****5678",
      "amount": 1500.00,
      "currency": "RUB",
      "status": "COMPLETED",
      "transferType": "ON_US",
      "fee": 0.00,
      "createdAt": "2026-04-28T14:30:00Z",
      "completedAt": "2026-04-28T14:30:05Z"
    }
  ],
  "page": 1,
  "size": 20,
  "totalElements": 156,
  "totalPages": 8
}
```

### 2.3. REST API — Детали перевода

```typescript
GET /api/v1/history/{transferId}

Response:
{
  "transferId": "UUID",
  "userId": "UUID",
  "fromCard": {
    "maskedNumber": "****1234",
    "bankName": "Bank A",
    "cardType": "VISA"
  },
  "toCard": {
    "maskedNumber": "****5678",
    "bankName": "Bank B",
    "cardType": "MASTERCARD"
  },
  "amount": 1500.00,
  "currency": "RUB",
  "exchangeRate": null,
  "originalAmount": null,
  "originalCurrency": null,
  "status": "COMPLETED",
  "transferType": "ON_US",
  "fee": 0.00,
  "feeCurrency": "RUB",
  "comment": "Перевод другу",
  "createdAt": "2026-04-28T14:30:00Z",
  "completedAt": "2026-04-28T14:30:05Z",
  "failedAt": null,
  "failureReason": null
}
```

### 2.4. REST API — Экспорт истории

```typescript
// Экспорт в CSV
GET /api/v1/history/export/csv?userId={UUID}&from=2026-01-01&to=2026-04-30
Response: Content-Type: text/csv

// Экспорт в PDF
GET /api/v1/history/export/pdf?userId={UUID}&from=2026-01-01&to=2026-04-30
Response: Content-Type: application/pdf

// Экспорт в Excel
GET /api/v1/history/export/xlsx?userId={UUID}&from=2026-01-01&to=2026-04-30
Response: Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet
```

### 2.5. REST API — Поиск по истории

```typescript
// Поиск по сумме, карте, комментарию
GET /api/v1/history/search?userId={UUID}&query=1500&field=amount&page=1&size=20

// Поиск по маскированному номеру карты
GET /api/v1/history/search?userId={UUID}&query=****1234&field=cardNumber

// Поиск по комментарию
GET /api/v1/history/search?userId={UUID}&query=другу&field=comment
```

### 2.6. REST API — Административные

```typescript
// Получение статистики по переводам
GET /api/v1/admin/history/stats?from=2026-01-01&to=2026-04-30
Response:
{
  "totalTransfers": 150000,
  "totalAmount": 750000000.00,
  "byStatus": {
    "COMPLETED": 145000,
    "FAILED": 3000,
    "CANCELLED": 2000
  },
  "byType": {
    "ON_US": 80000,
    "OFF_US": 40000,
    "ME_TO_ME": 20000,
    "SBP": 10000
  },
  "byCurrency": {
    "RUB": 140000,
    "USD": 7000,
    "EUR": 3000
  }
}

// Очистка истории пользователя (GDPR)
DELETE /api/v1/admin/history/{userId}
```

---

## 3. Валидации

### 3.1. Структурные валидации

| Поле | Проверка | Ошибка |
|------|----------|--------|
| `userId` | Валидный UUID | INVALID_USER_ID |
| `transferId` | Валидный UUID | INVALID_TRANSFER_ID |
| `page` | >= 1 | INVALID_PAGE |
| `size` | 1-100 | INVALID_PAGE_SIZE |
| `from` | <= to | INVALID_DATE_RANGE |
| `status` | Одно из допустимых значений | INVALID_STATUS |
| `currency` | Существует в справочнике | UNSUPPORTED_CURRENCY |

### 3.2. Бизнес-валидации

**3.2.1. Доступ к истории:**
- Пользователь может видеть только свои переводы
- Администратор может видеть все переводы
- При запросе чужого transferId → 403 Forbidden

**3.2.2. Ограничения экспорта:**
- Максимальный период экспорта: 1 год
- Максимальное количество записей в экспорте: 10000
- Экспорт доступен не чаще 1 раза в 5 минут

**3.2.3. Хранение данных:**
- История хранится 5 лет (согласно требованиям регулятора)
- После 5 лет — архивация и удаление из основной БД
- По запросу пользователя (GDPR) — полное удаление истории

---

## 4. Основная логика

### 4.1. Процесс сохранения события (HistoryEventProcessor)

```
ШАГ 1: ПОЛУЧЕНИЕ СОБЫТИЯ ИЗ KAFKA
  ├── Десериализация события
  ├── Проверка idempotency key (transferId)
  ├── Если уже существует → обновление статуса
  └── Если новое → создание записи

ШАГ 2: СОХРАНЕНИЕ В БД
  ├── INSERT INTO transfer_history (transferId, userId, ...)
  ├── Если событие TransferCompleted:
  │   └── UPDATE SET status = 'COMPLETED', completedAt = NOW()
  ├── Если событие TransferFailed:
  │   └── UPDATE SET status = 'FAILED', failedAt = NOW(), failureReason = ...
  └── Если событие TransferCancelled:
      └── UPDATE SET status = 'CANCELLED'

ШАГ 3: ОБНОВЛЕНИЕ КЭША
  ├── Обновление Redis: последние 50 переводов пользователя
  └── TTL кэша: 1 час
```

### 4.2. Процесс получения истории (HistoryQueryService)

```
ШАГ 1: ПАРСИНГ ПАРАМЕТРОВ ЗАПРОСА
  ├── Валидация userId
  ├── Парсинг фильтров (status, currency, date range)
  ├── Парсинг пагинации (page, size)
  └── Парсинг сортировки (sortBy, sortDir)

ШАГ 2: ПОИСК В КЭШЕ (только для первых 50 записей)
  ├── Если page = 1 и size <= 50 и нет фильтров:
  │   ├── Поиск в Redis
  │   └── Если найдено → возврат из кэша
  └── Иначе → запрос к БД

ШАГ 3: ЗАПРОС К БД
  ├── SELECT с фильтрацией, пагинацией, сортировкой
  ├── Оптимизированный запрос с использованием индексов
  └── Возврат Page<TransferHistory>

ШАГ 4: МАСКИРОВАНИЕ ДАННЫХ
  ├── Маскирование номеров карт (****1234)
  ├── Маскирование персональных данных
  └── Возврат DTO
```

### 4.3. Процесс экспорта истории (HistoryExportService)

```
ШАГ 1: ВАЛИДАЦИЯ ЗАПРОСА
  ├── Проверка периода (max 1 год)
  ├── Проверка количества записей (max 10000)
  └── Проверка rate limit (1 раз в 5 минут)

ШАГ 2: ЗАГРУЗКА ДАННЫХ
  ├── SELECT с фильтрацией по периоду
  ├── Batch loading (по 1000 записей)
  └── Если > 10000 записей → ошибка

ШАГ 3: ГЕНЕРАЦИЯ ФАЙЛА
  ├── CSV: построчная запись с заголовками
  ├── PDF: таблица с форматированием
  └── Excel: форматированная таблица с автофильтрами

ШАГ 4: ВОЗВРАТ ФАЙЛА
  ├── Stream response (не загружать в память целиком)
  ├── Content-Type и Content-Disposition заголовки
  └── Логирование факта экспорта
```

### 4.4. Процесс поиска по истории (HistorySearchService)

```
ШАГ 1: ОПРЕДЕЛЕНИЕ ПОЛЯ ПОИСКА
  ├── amount → поиск по точной сумме или диапазону
  ├── cardNumber → поиск по маскированному номеру
  ├── comment → полнотекстовый поиск (ILIKE)
  └── transferId → точный поиск

ШАГ 2: ВЫПОЛНЕНИЕ ПОИСКА
  ├── Для amount: WHERE amount = :query
  ├── Для cardNumber: WHERE fromCard LIKE :query OR toCard LIKE :query
  ├── Для comment: WHERE comment ILIKE '%' || :query || '%'
  └── Для transferId: WHERE transferId = :query

ШАГ 3: ВОЗВРАТ РЕЗУЛЬТАТА
  └── Page<TransferHistory> с пагинацией
```

---

## 5. Интеграции

### 5.1. Внутренние сервисы

| Сервис | Протокол | Метод | Назначение |
|--------|----------|-------|------------|
| **Transfer Service** | Kafka | Consumer | Получение событий переводов |
| **Mobile App** | REST | `GET /api/v1/history` | Получение истории |
| **Web App** | REST | `GET /api/v1/history` | Получение истории |
| **Admin Panel** | REST | `GET /api/v1/admin/history/*` | Статистика и управление |

### 5.2. Kafka

**Топик:** `banking.transfer.events` (Consumer)

```yaml
Группа: history-service-group
Client ID: history-service
Auto Commit: false
Max Poll Records: 100
```

### 5.3. База данных

```sql
-- Основная таблица истории переводов
CREATE TABLE transfer_history (
    id UUID PRIMARY KEY,
    transfer_id UUID NOT NULL UNIQUE,
    user_id UUID NOT NULL,
    from_card_masked VARCHAR(8) NOT NULL,       -- ****1234
    from_card_bank VARCHAR(100),
    from_card_type VARCHAR(20),                  -- VISA, MASTERCARD, MIR
    to_card_masked VARCHAR(8) NOT NULL,          -- ****5678
    to_card_bank VARCHAR(100),
    to_card_type VARCHAR(20),
    amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    exchange_rate DECIMAL(10,6),
    original_amount DECIMAL(10,2),
    original_currency VARCHAR(3),
    status VARCHAR(20) NOT NULL,                 -- PENDING, COMPLETED, FAILED, CANCELLED
    transfer_type VARCHAR(20) NOT NULL,          -- ON_US, OFF_US, ME_TO_ME, SBP
    fee DECIMAL(10,2) DEFAULT 0.00,
    fee_currency VARCHAR(3),
    comment VARCHAR(200),
    failure_reason VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    failed_at TIMESTAMP,
    cancelled_at TIMESTAMP
);

-- Таблица для архивации (переводы старше 5 лет)
CREATE TABLE transfer_history_archive (
    LIKE transfer_history INCLUDING ALL,
    archived_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Партиционирование по месяцам (для производительности)
CREATE TABLE transfer_history_2026_01 PARTITION OF transfer_history
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE transfer_history_2026_02 PARTITION OF transfer_history
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- Индексы
CREATE INDEX idx_transfer_history_user_id ON transfer_history(user_id);
CREATE INDEX idx_transfer_history_user_id_created ON transfer_history(user_id, created_at DESC);
CREATE INDEX idx_transfer_history_status ON transfer_history(status);
CREATE INDEX idx_transfer_history_transfer_type ON transfer_history(transfer_type);
CREATE INDEX idx_transfer_history_currency ON transfer_history(currency);
CREATE INDEX idx_transfer_history_created_at ON transfer_history(created_at);
CREATE INDEX idx_transfer_history_comment_search ON transfer_history USING gin(to_tsvector('russian', comment));
CREATE INDEX idx_transfer_history_amount ON transfer_history(amount);

-- Таблица для отслеживания экспортов (rate limiting)
CREATE TABLE export_log (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    export_type VARCHAR(10) NOT NULL,            -- CSV, PDF, XLSX
    format VARCHAR(10) NOT NULL,
    record_count INTEGER NOT NULL,
    file_size_bytes BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_export_log_user_id_created ON export_log(user_id, created_at);
```

---

## 6. Исключительные ситуации

### 6.1. Валидационные ошибки (HTTP 400)

| Код ошибки | Сообщение | Условие |
|------------|-----------|---------|
| `INVALID_USER_ID` | "Некорректный идентификатор пользователя" | userId не UUID |
| `INVALID_TRANSFER_ID` | "Некорректный идентификатор перевода" | transferId не UUID |
| `INVALID_PAGE` | "Некорректный номер страницы" | page < 1 |
| `INVALID_PAGE_SIZE` | "Некорректный размер страницы" | size < 1 или > 100 |
| `INVALID_DATE_RANGE` | "Некорректный диапа

| `INVALID_DATE_RANGE` | "Некорректный диапазон дат" | from > to |
| `INVALID_STATUS` | "Некорректный статус" | status не из допустимого списка |
| `UNSUPPORTED_CURRENCY` | "Валюта не поддерживается" | currency отсутствует в справочнике |
| `INVALID_SORT_FIELD` | "Некорректное поле сортировки" | sortBy не из допустимого списка |

### 6.2. Бизнес-ошибки (HTTP 422)

| Код ошибки | Сообщение | Условие |
|------------|-----------|---------|
| `TRANSFER_NOT_FOUND` | "Перевод не найден" | transferId не существует |
| `FORBIDDEN_ACCESS` | "Нет доступа к истории перевода" | Попытка доступа к чужому переводу |
| `EXPORT_PERIOD_TOO_LARGE` | "Период экспорта превышает 1 год" | from - to > 365 дней |
| `EXPORT_TOO_MANY_RECORDS` | "Превышено количество записей для экспорта" | > 10000 записей |
| `EXPORT_RATE_LIMIT` | "Экспорт доступен не чаще 1 раза в 5 минут" | Последний экспорт < 5 минут назад |
| `HISTORY_ALREADY_DELETED` | "История уже удалена" | GDPR delete уже выполнен |
| `HISTORY_NOT_FOUND` | "История не найдена" | Нет переводов за указанный период |

### 6.3. Интеграционные ошибки (HTTP 502/503)

| Код ошибки | Сообщение | Условие | Действие |
|------------|-----------|---------|----------|
| `DATABASE_TIMEOUT` | "Таймаут при запросе к БД" | PostgreSQL не отвечает > 3 сек | Retry 2 раза, затем ошибка |
| `DATABASE_CONNECTION` | "Ошибка подключения к БД" | Connection pool исчерпан | Retry с exponential backoff |
| `KAFKA_CONSUMER_ERROR` | "Ошибка обработки события Kafka" | Ошибка десериализации | Отправка в DLQ |
| `CACHE_UNAVAILABLE` | "Кэш недоступен" | Redis не отвечает | Graceful degradation (чтение из БД) |
| `EXPORT_GENERATION_ERROR` | "Ошибка генерации файла экспорта" | Ошибка Apache POI / iText | Retry 1 раз |

### 6.4. Стратегии восстановления

**6.4.1. Транзакционные откаты:**
- При ошибке сохранения события → автоматический ROLLBACK
- При ошибке обновления статуса → повторная попытка через Kafka retry

**6.4.2. Graceful degradation:**
- При недоступности Redis → чтение напрямую из PostgreSQL
- При недоступности Kafka → синхронное сохранение через REST (если Transfer Service вызывает напрямую)
- При ошибке генерации экспорта → повторная попытка через 1 минуту

**6.4.3. Обработка больших объёмов:**
- Партиционирование таблицы по месяцам
- Архивация записей старше 5 лет в отдельную таблицу
- Фоновый scheduler для очистки и архивации (еженедельно)

---

## 7. Выходные данные

### 7.1. REST API — Успешные ответы

**7.1.1. История переводов (200):**
```json
{
  "content": [
    {
      "transferId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "fromCard": "****1234",
      "toCard": "****5678",
      "amount": 1500.00,
      "currency": "RUB",
      "status": "COMPLETED",
      "transferType": "ON_US",
      "fee": 0.00,
      "createdAt": "2026-04-28T14:30:00Z",
      "completedAt": "2026-04-28T14:30:05Z"
    }
  ],
  "page": 1,
  "size": 20,
  "totalElements": 156,
  "totalPages": 8,
  "first": true,
  "last": false
}
```

**7.1.2. Детали перевода (200):**
```json
{
  "transferId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "userId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "fromCard": {
    "maskedNumber": "****1234",
    "bankName": "АО 'Банк А'",
    "cardType": "VISA"
  },
  "toCard": {
    "maskedNumber": "****5678",
    "bankName": "АО 'Банк Б'",
    "cardType": "MASTERCARD"
  },
  "amount": 1500.00,
  "currency": "RUB",
  "exchangeRate": null,
  "originalAmount": null,
  "originalCurrency": null,
  "status": "COMPLETED",
  "transferType": "ON_US",
  "fee": 0.00,
  "feeCurrency": "RUB",
  "comment": "Перевод другу",
  "createdAt": "2026-04-28T14:30:00Z",
  "completedAt": "2026-04-28T14:30:05Z",
  "failedAt": null,
  "failureReason": null
}
```

**7.1.3. Статистика (200):**
```json
{
  "period": { "from": "2026-01-01", "to": "2026-04-30" },
  "totalTransfers": 150000,
  "totalAmount": 750000000.00,
  "averageAmount": 5000.00,
  "byStatus": {
    "COMPLETED": { "count": 145000, "amount": 725000000.00 },
    "FAILED": { "count": 3000, "amount": 15000000.00 },
    "CANCELLED": { "count": 2000, "amount": 10000000.00 }
  },
  "byType": {
    "ON_US": { "count": 80000, "amount": 400000000.00 },
    "OFF_US": { "count": 40000, "amount": 200000000.00 },
    "ME_TO_ME": { "count": 20000, "amount": 100000000.00 },
    "SBP": { "count": 10000, "amount": 50000000.00 }
  },
  "byCurrency": {
    "RUB": { "count": 140000, "amount": 700000000.00 },
    "USD": { "count": 7000, "amount": 35000000.00 },
    "EUR": { "count": 3000, "amount": 15000000.00 }
  }
}
```

### 7.2. Форматы экспорта

**7.2.1. CSV формат:**
```csv
transferId,fromCard,toCard,amount,currency,status,transferType,fee,createdAt,completedAt
uuid,****1234,****5678,1500.00,RUB,COMPLETED,ON_US,0.00,2026-04-28T14:30:00Z,2026-04-28T14:30:05Z
```

**7.2.2. PDF формат:**
- Заголовок: "История переводов за период"
- Таблица с колонками: Дата, С карты, На карту, Сумма, Валюта, Статус, Комиссия
- Итоговая строка с суммой всех переводов
- Нумерация страниц

**7.2.3. Excel формат:**
- Лист "История переводов" с форматированной таблицей
- Автофильтры на каждом столбце
- Цветовая индикация статусов (зелёный = COMPLETED, красный = FAILED)
- Итоговая строка с SUM и COUNT

---

## 8. Производительность

### 8.1. Целевые показатели

| Метрика | Целевое значение | Метод измерения |
|---------|-----------------|-----------------|
| Время получения истории | ≤ 200 мс (p99) | JMeter, Gatling |
| Время получения деталей | ≤ 100 мс (p99) | JMeter, Gatling |
| Время поиска | ≤ 500 мс (p99) | JMeter, Gatling |
| Время экспорта CSV | ≤ 5 сек (10000 записей) | Load testing |
| Время экспорта PDF | ≤ 10 сек (10000 записей) | Load testing |
| Время экспорта Excel | ≤ 10 сек (10000 записей) | Load testing |
| Пропускная способность (чтение) | 5000 RPS | Load testing |
| Пропускная способность (запись) | 1000 RPS | Load testing |

### 8.2. Оптимизации

**8.2.1. Кэширование:**
- **Redis Cache:** последние 50 переводов пользователя (TTL: 1 час)
- **Redis Cache:** статистика по переводам (TTL: 15 минут)
- **Local Cache (Caffeine):** справочные данные (TTL: 10 минут)

**8.2.2. Индексы БД:**
- `(user_id, created_at DESC)` — основной запрос истории
- `(user_id, status, created_at DESC)` — фильтрация по статусу
- `(user_id, currency, created_at DESC)` — фильтрация по валюте
- `(created_at)` — для фоновой архивации
- GIN индекс на `to_tsvector('russian', comment)` — полнотекстовый поиск

**8.2.3. Партиционирование:**
- Таблица `transfer_history` партиционирована по месяцам
- Автоматическое создание новых партиций (scheduler)
- Старые партиции (> 5 лет) перемещаются в архив

**8.2.4. Пагинация:**
- Offset-based пагинация для страниц до 100
- Cursor-based пагинация для глубоких страниц (> 100)
- Максимальный offset: 10000 (защита от полного сканирования)

### 8.3. Ограничения

| Параметр | Значение | Обоснование |
|----------|----------|-------------|
| Максимальная нагрузка (чтение) | 10000 RPS | Пиковые нагрузки |
| Максимальная нагрузка (запись) | 2000 RPS | Пиковые нагрузки |
| Максимальный период хранения | 5 лет | Требования регулятора |
| Максимальный размер БД | 10 TB | С учётом роста на 5 лет |
| Максимальный размер ответа | 1 MB | Ограничение API Gateway |
| Максимальный offset пагинации | 10000 | Защита производительности |

---

## 9. Безопасность

### 9.1. Аутентификация и авторизация

| Endpoint | Метод | Роль | Аутентификация |
|----------|-------|------|----------------|
| `GET /api/v1/history` | Пользовательский | USER | JWT + userId match |
| `GET /api/v1/history/{transferId}` | Пользовательский | USER | JWT + владелец перевода |
| `GET /api/v1/history/search` | Пользовательский | USER | JWT + userId match |
| `GET /api/v1/history/export/*` | Пользовательский | USER | JWT + userId match |
| `GET /api/v1/admin/history/*` | Административный | ADMIN | JWT + Role check |
| `DELETE /api/v1/admin/history/{userId}` | Административный | ADMIN | JWT + Role check |

### 9.2. Защита данных

- **PCI DSS:** номера карт хранятся только в маскированном виде (****1234)
- **GDPR:** полное удаление истории по запросу пользователя
- **Шифрование:** данные в покое зашифрованы (AES-256)
- **Маскирование:** номера карт, телефоны, email маскируются в ответах API
- **Логирование:** без записи персональных данных (PII) в логи
- **Аудит:** все запросы к истории логируются (кто, когда, что запрашивал)

### 9.3. Rate Limiting

| Endpoint | Лимит | Период |
|----------|-------|--------|
| `GET /api/v1/history` | 100 запросов | 1 минута |
| `GET /api/v1/history/{transferId}` | 100 запросов | 1 минута |
| `GET /api/v1/history/search` | 50 запросов | 1 минута |
| `GET /api/v1/history/export/*` | 5 запросов | 1 минута |
| `GET /api/v1/admin/history/*` | 30 запросов | 1 минута |
| `DELETE /api/v1/admin/history/*` | 10 запросов | 1 минута |

### 9.4. GDPR Compliance

- Пользователь может запросить удаление всей своей истории
- Удаление выполняется в течение 30 дней
- После удаления — запись в audit log
- Невозможно восстановить удалённые данные
- Экспорт данных предоставляется в машиночитаемом формате (JSON)