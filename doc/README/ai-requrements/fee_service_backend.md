# Системные требования: Fee Service (Микросервис тарификации и комиссий)

**Версия:** 1.0  
**Дата:** 24.04.2026  
**Связь с архитектурой:** [`c4_Level_2_containers_diagram_Card2CardTransfer_v1.plantuml`](c4_Level_2_containers_diagram_Card2CardTransfer_v1.plantuml)  
**Связь с компонентами:** [`c4_Level_3_component_diagram_Card2CardTransfer_(FeeService)_v1.plantuml`](c4_Level_3_component_diagram_Card2CardTransfer_(FeeService)_v1.plantuml)  
**Связь с BRD:** [`card2card_transfer_brd.md`](card2card_transfer_brd.md)  
**Связь с Transfer Service:** [`transfer_service_backend.md`](transfer_service_backend.md)  

---

## 1. Общий обзор

### 1.1. Назначение
Fee Service — микросервис для расчета комиссий и управления тарифной политикой переводов с карты на карту. Сервис отвечает за:
- Расчет комиссии для каждого перевода на основе тарифных правил
- Управление тарифными планами (создание, редактирование, деактивация)
- Применение акций и скидок на комиссии
- Аудит всех расчетов комиссий
- Интеграцию с Transfer Service для синхронного расчета комиссии

### 1.2. Высокоуровневая логика
```
Получение запроса → Определение типа перевода → Поиск тарифа → 
Проверка акций → Расчет комиссии → Логирование → Возврат результата
```

### 1.3. Технологический стек
- **Язык:** Java 17+
- **Фреймворк:** Spring Boot 3.x
- **БД:** PostgreSQL (Fee Database)
- **Очередь:** Kafka Consumer (подписка на события переводов)
- **Интеграции:** REST (Feign Client), Kafka Consumer

---

## 2. Входные данные

### 2.1. REST API — Расчет комиссии

```typescript
POST /api/v1/fees/calculate
Request Body:
{
  "sourceCardId": "UUID",              // ID карты отправителя (обязательно)
  "targetCardNumber": "string",        // Номер карты получателя (обязательно)
  "amount": "number",                  // Сумма перевода > 0 (обязательно)
  "currency": "string",                // Код валюты ISO 4217 (обязательно)
  "transferType": "ON_US | OFF_US | ME_TO_ME | SBP",  // Тип перевода (обязательно)
  "userId": "UUID"                     // ID пользователя (обязательно)
}
```

### 2.2. REST API — Управление тарифами (административные)

```typescript
// Создание тарифа
POST /api/v1/admin/tariffs
Request Body:
{
  "name": "string",                    // Название тарифа (обязательно)
  "description": "string",             // Описание (опционально)
  "transferType": "ON_US | OFF_US | ME_TO_ME | SBP",  // Тип перевода (обязательно)
  "feeType": "PERCENTAGE | FIXED | MIXED",  // Тип комиссии (обязательно)
  "feeValue": "number",                // Значение комиссии (обязательно)
  "minFee": "number",                  // Минимальная комиссия (опционально)
  "maxFee": "number",                  // Максимальная комиссия (опционально)
  "currency": "string",                // Валюта тарифа (обязательно)
  "cardBins": ["string"],              // BIN-диапазоны карт (опционально)
  "amountFrom": "number",              // Нижняя граница суммы (опционально)
  "amountTo": "number",                // Верхняя граница суммы (опционально)
  "priority": "integer",               // Приоритет правила (обязательно)
  "isActive": "boolean"                // Активность (обязательно)
}

// Редактирование тарифа
PUT /api/v1/admin/tariffs/{tariffId}

// Деактивация тарифа
DELETE /api/v1/admin/tariffs/{tariffId}

// Получение списка тарифов
GET /api/v1/admin/tariffs?page=0&size=20&activeOnly=true
```

### 2.3. REST API — Управление акциями

```typescript
// Создание акции
POST /api/v1/admin/promotions
Request Body:
{
  "name": "string",                    // Название акции (обязательно)
  "description": "string",             // Описание (опционально)
  "discountType": "PERCENTAGE | FIXED",  // Тип скидки (обязательно)
  "discountValue": "number",           // Значение скидки (обязательно)
  "tariffIds": ["UUID"],               // ID тарифов, на которые действует скидка
  "userId": "UUID",                    // ID пользователя (для персональных акций)
  "cardBins": ["string"],              // BIN-диапазоны (опционально)
  "startDate": "datetime",             // Дата начала (обязательно)
  "endDate": "datetime",               // Дата окончания (обязательно)
  "maxUsageCount": "integer",          // Максимальное количество использований
  "isActive": "boolean"                // Активность (обязательно)
}
```

### 2.4. Kafka Consumer — События переводов

```yaml
Топик: banking.transfer.events
Группа: fee-service-group
Типы событий:
  - TransferInitiated  # Для асинхронного логирования комиссии
  - TransferCompleted  # Для фиксации фактической комиссии
  - TransferFailed     # Для отмены резервирования комиссии
```

---

## 3. Валидации

### 3.1. Структурные валидации

| Поле | Проверка | Ошибка |
|------|----------|--------|
| `sourceCardId` | Валидный UUID | INVALID_CARD_ID |
| `targetCardNumber` | 16-19 цифр, алгоритм Луна | INVALID_CARD_NUMBER |
| `amount` | > 0 и <= 9999999.99 | INVALID_AMOUNT |
| `currency` | Существует в справочнике | UNSUPPORTED_CURRENCY |
| `transferType` | Одно из: ON_US, OFF_US, ME_TO_ME, SBP | INVALID_TRANSFER_TYPE |
| `feeValue` | > 0 для FIXED, 0-100 для PERCENTAGE | INVALID_FEE_VALUE |

### 3.2. Бизнес-валидации

**3.2.1. Проверка тарифа:**
- Тариф существует и активен (isActive = true)
- Дата действия тарифа не истекла
- Сумма перевода попадает в диапазон [amountFrom, amountTo]
- BIN карты получателя попадает в диапазон cardBins (если указан)

**3.2.2. Проверка акции:**
- Акция существует и активна
- Дата в пределах [startDate, endDate]
- Лимит использований не превышен (maxUsageCount)
- Акция применима к данному пользователю/карте

**3.2.3. Проверка me-to-me:**
- Если transferType = ME_TO_ME → комиссия = 0 (бесплатно)
- Не требуется проверка тарифов

### 3.3. Административные валидации

**3.3.1. Конфликт тарифов:**
- При создании тарифа проверка на пересечение с существующими
- Одинаковый transferType + пересекающиеся BIN-диапазоны → конфликт
- Разрешение конфликта через priority (выигрывает больший priority)

**3.3.2. Валидация акций:**
- startDate < endDate
- discountValue > 0
- Для PERCENTAGE: discountValue <= 100

---

## 4. Основная логика

### 4.1. Процесс расчета комиссии (FeeCalculator)

```
ШАГ 1: ОПРЕДЕЛЕНИЕ ТИПА ПЕРЕВОДА
  ├── Если transferType = ME_TO_ME → комиссия = 0
  └── Иначе → переход к шагу 2

ШАГ 2: ПОИСК ПРИМЕНИМОГО ТАРИФА (FeeRulesEngine)
  ├── Поиск тарифов по transferType
  ├── Фильтрация по BIN карты получателя
  ├── Фильтрация по сумме перевода
  ├── Сортировка по priority (DESC)
  └── Выбор тарифа с наивысшим priority

ШАГ 3: РАСЧЕТ БАЗОВОЙ КОМИССИИ
  ├── Если feeType = FIXED:
  │   └── fee = feeValue
  ├── Если feeType = PERCENTAGE:
  │   └── fee = amount * feeValue / 100
  └── Если feeType = MIXED:
      └── fee = fixedPart + amount * percentagePart / 100

ШАГ 4: ПРИМЕНЕНИЕ ОГРАНИЧЕНИЙ
  ├── Если minFee задан и fee < minFee → fee = minFee
  └── Если maxFee задан и fee > maxFee → fee = maxFee

ШАГ 5: ПРОВЕРКА АКЦИЙ (PromotionEngine)
  ├── Поиск активных акций для данного пользователя/карты
  ├── Если акция найдена:
  │   ├── PERCENTAGE: fee = fee * (1 - discountValue / 100)
  │   └── FIXED: fee = fee - discountValue
  └── Если fee < 0 → fee = 0

ШАГ 6: ЛОГИРОВАНИЕ (FeeAuditService)
  ├── Сохранение записи в fee_log
  ├── Поля: transferId, userId, amount, fee, tariffId, promotionId
  └── Публикация события: FeeCalculated

ШАГ 7: ВОЗВРАТ РЕЗУЛЬТАТА
  └── { fee, tariffName, promotionName, calculationDetails }
```

### 4.2. Движок правил тарификации (FeeRulesEngine)

```javascript
function findApplicableTariff(transferType, cardBIN, amount) {
  // 1. Получение всех активных тарифов для данного типа перевода
  const tariffs = tariffRepository.findByTransferTypeAndIsActiveTrue(transferType);
  
  // 2. Фильтрация по BIN
  const binTariffs = tariffs.filter(t => {
    if (!t.cardBins || t.cardBins.length === 0) return true;
    return t.cardBins.some(binRange => isBINInRange(cardBIN, binRange));
  });
  
  // 3. Фильтрация по сумме
  const amountTariffs = binTariffs.filter(t => {
    if (t.amountFrom && amount < t.amountFrom) return false;
    if (t.amountTo && amount > t.amountTo) return false;
    return true;
  });
  
  // 4. Сортировка по priority (DESC) и выбор первого
  amountTariffs.sort((a, b) => b.priority - a.priority);
  
  return amountTariffs[0] || null; // null = тариф не найден
}
```

### 4.3. Управление акциями (PromotionEngine)

```javascript
function findApplicablePromotion(userId, cardBIN, tariffId) {
  const now = new Date();
  
  // 1. Поиск активных акций
  const promotions = promotionRepository.findActivePromotions(now);
  
  // 2. Фильтрация по тарифу
  const tariffPromotions = promotions.filter(p => 
    !p.tariffIds || p.tariffIds.length === 0 || p.tariffIds.includes(tariffId)
  );
  
  // 3. Фильтрация по пользователю
  const userPromotions = tariffPromotions.filter(p =>
    !p.userId || p.userId === userId
  );
  
  // 4. Фильтрация по BIN
  const binPromotions = userPromotions.filter(p =>
    !p.cardBins || p.cardBins.length === 0 || 
    p.cardBins.some(binRange => isBINInRange(cardBIN, binRange))
  );
  
  // 5. Проверка лимита использований
  const availablePromotions = binPromotions.filter(p => {
    if (!p.maxUsageCount) return true;
    const usageCount = feeLogRepository.countByPromotionId(p.id);
    return usageCount < p.maxUsageCount;
  });
  
  // 6. Выбор акции с максимальной скидкой
  availablePromotions.sort((a, b) => b.discountValue - a.discountValue);
  
  return availablePromotions[0] || null;
}
```

---

## 5. Интеграции

### 5.1. Внутренние сервисы

| Сервис | Протокол | Метод | Назначение |
|--------|----------|-------|------------|
| **Transfer Service** | REST (Feign Client) | `POST /api/v1/fees/calculate` | Синхронный расчет комиссии |
| **Limit Service** | REST (Feign Client) | `GET /api/v1/limits/check` | Проверка лимитов для корректировки комиссии |
| **Admin Panel** | REST | `GET/POST/PUT/DELETE /api/v1/admin/*` | Управление тарифами и акциями |

### 5.2. Kafka

**Топик:** `banking.transfer.events` (Consumer)

```yaml
Группа: fee-service-group
Client ID: fee-service
Auto Commit: false
Max Poll Records: 100
```

**Обработка событий:**

| Событие | Действие |
|---------|----------|
| `TransferInitiated` | Логирование предварительного расчета комиссии |
| `TransferCompleted` | Фиксация фактической комиссии в истории |
| `TransferFailed` | Отмена резервирования комиссии (если применимо) |

### 5.3. База данных

```sql
-- Таблица тарифных планов
CREATE TABLE tariffs (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    transfer_type VARCHAR(20) NOT NULL,  -- ON_US, OFF_US, ME_TO_ME, SBP
    fee_type VARCHAR(20) NOT NULL,        -- PERCENTAGE, FIXED, MIXED
    fee_value DECIMAL(10,4) NOT NULL,
    min_fee DECIMAL(10,2),
    max_fee DECIMAL(10,2),
    currency VARCHAR(3) NOT NULL,
    card_bins JSONB,                      -- Массив BIN-диапазонов
    amount_from DECIMAL(10,2),
    amount_to DECIMAL(10,2),
    priority INTEGER NOT NULL DEFAULT 0,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица акций
CREATE TABLE promotions (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    discount_type VARCHAR(20) NOT NULL,   -- PERCENTAGE, FIXED
    discount_value DECIMAL(10,4) NOT NULL,
    tariff_ids UUID[],                    -- Массив ID тарифов
    user_id UUID,                         -- NULL = для всех пользователей
    card_bins JSONB,
    start_date TIMESTAMP NOT NULL,
    end_date TIMESTAMP NOT NULL,
    max_usage_count INTEGER,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица логов комиссий
CREATE TABLE fee_logs (
    id UUID PRIMARY KEY,
    transfer_id UUID NOT NULL,
    user_id UUID NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    fee DECIMAL(10,2) NOT NULL,
    tariff_id UUID,
    promotion_id UUID,
    calculation_details JSONB,            -- Детали расчета
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Индексы
CREATE INDEX idx_tariffs_transfer_type ON tariffs(transfer_type);
CREATE INDEX idx_tariffs_is_active ON tariffs(is_active);
CREATE INDEX idx_promotions_dates ON promotions(start_date, end_date);
CREATE INDEX idx_fee_logs_transfer_id ON fee_logs(transfer_id);
CREATE INDEX idx_fee_logs_user_id ON fee_logs(user_id);
```

---

## 6. Исключительные ситуации

### 6.1. Валидационные ошибки (HTTP 400)

| Код ошибки | Сообщение | Условие |
|------------|-----------|---------|
| `INVALID_TRANSFER_TYPE` | "Некорректный тип перевода" | transferType не из списка ON_US, OFF_US, ME_TO_ME, SBP |
| `INVALID_FEE_VALUE` | "Некорректное значение комиссии" | feeValue вне допустимого диапазона |
| `INVALID_DISCOUNT_VALUE` | "Некорректное значение скидки" | discountValue <= 0 или > 100 для PERCENTAGE |
| `INVALID_DATE_RANGE` | "Дата начала должна быть раньше даты окончания" | startDate >= endDate |

### 6.2. Бизнес-ошибки (HTTP 422)

| Код ошибки | Сообщение | Условие |
|------------|-----------|---------|
| `TARIFF_NOT_FOUND` | "Не найден подходящий тариф для данного перевода" | Нет активного тарифа для transferType |
| `TARIFF_CONFLICT` | "Обнаружен конфликт тарифов. Разрешите через priority" | Пересечение BIN-диапазонов с одинаковым priority |
| `PROMOTION_NOT_FOUND` | "Акция не найдена или неактивна" | Акция не существует или isActive = false |
| `PROMOTION_EXPIRED` | "Срок действия акции истек" | Текущая дата вне [startDate, endDate] |
| `PROMOTION_LIMIT_EXCEEDED` | "Лимит использований акции исчерпан" | usageCount >= maxUsageCount |

### 6.3. Интеграционные ошибки (HTTP 502/503)

| Код ошибки | Сообщение | Условие |
|------------|-----------|---------|
| `LIMIT_SERVICE_UNAVAILABLE` | "Сервис лимитов недоступен. Комиссия рассчитана без учета лимитов" | Limit Service не отвечает |
| `TRANSFER_SERVICE_UNAVAILABLE` | "Сервис переводов недоступен" | Transfer Service не отвечает |

### 6.4. Стратегии восстановления

| Тип ошибки | Стратегия | Действие |
|------------|-----------|----------|
| **Тариф не найден** | Default tariff | Применение тарифа по умолчанию (default_fee) |
| **Ошибка БД** | Retry (3 попытки) | Exponential backoff: 500ms, 1s, 2s |
| **Недоступность Limit Service** | Graceful degradation | Расчет комиссии без учета лимитов |
| **Конфликт данных** | Optimistic locking | Повторная попытка с обновленными данными |

---

## 7. Выходные данные

### 7.1. Успешный ответ расчета комиссии (HTTP 200)

```json
{
  "fee": 15.00,
  "currency": "RUB",
  "tariffName": "Стандартный тариф off-us",
  "tariffDescription": "1% от суммы перевода, макс. 100 ₽",
  "promotionName": "Акция: первая комиссия бесплатно",
  "promotionDiscount": 15.00,
  "calculationDetails": {
    "baseFee": 15.00,
    "minFee": null,
    "maxFee": 100.00,
    "appliedPromotion": true,
    "finalFee": 0.00
  }
}
```

### 7.2. Ошибка валидации (HTTP 400)

```json
{
  "error": "VALIDATION_ERROR",
  "code": "INVALID_TRANSFER_TYPE",
  "message": "Некорректный тип перевода",
  "details": {
    "field": "transferType",
    "value": "INVALID_TYPE",
    "allowedValues": ["ON_US", "OFF_US", "ME_TO_ME", "SBP"]
  }
}
```

### 7.3. Бизнес-ошибка (HTTP 422)

```json
{
  "error": "BUSINESS_ERROR",
  "code": "TARIFF_NOT_FOUND",
  "message": "Не найден подходящий тариф для данного перевода",
  "details": {
    "transferType": "OFF_US",
    "amount": 5000.00,
    "currency": "RUB",
    "cardBIN": "427655"
  }
}
```

---

## 8. Производительность

### 8.1. Целевые показатели

| Метрика | Целевое значение | Метод измерения |
|---------|-----------------|-----------------|
| Время расчета комиссии | ≤ 100 мс (p99) | APM мониторинг |
| Пропускная способность | ≥ 2000 запросов/сек | Нагрузочное тестирование |
| Время отклика API | ≤ 50 мс (p95) | APM мониторинг |
| Доступность сервиса | 99.9% | Health check мониторинг |

### 8.2. Оптимизации

- **Кэширование тарифов** в памяти сервиса (обновление при изменении через Kafka)
- **Кэширование BIN-справочника** (обновление раз в час)
- **Пул соединений к БД** (HikariCP, max: 10 соединений)
- **Индексы БД**: (transfer_type, is_active), (start_date, end_date), (transfer_id)

### 8.3. Ограничения

| Параметр | Значение |
|----------|----------|
| Максимальное количество тарифов | 1000 |
| Максимальное количество акций | 500 |
| Максимальная глубина BIN-диапазонов | 6 цифр |
| Таймаут расчета комиссии | 5 секунд |
| Размер пула потоков | 20 потоков |

---

## 9. Безопасность

### 9.1. Аутентификация и авторизация
- **Публичные endpoints** (`POST /api/v1/fees/calculate`): доступны только для Transfer Service (service-to-service аутентификация)
- **Административные endpoints** (`/api/v1/admin/*`): требуют JWT-токен с ролью ADMIN
- Rate limiting: 100 запросов в минуту для публичных, 30 для административных

### 9.2. Аудит
- Логирование всех изменений тарифов (кто, когда, что изменил)
- Логирование всех расчетов комиссий (transferId, userId, fee)
- Аудит применения акций (promotionId, userId, discount)