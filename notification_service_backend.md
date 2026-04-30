# Системные требования: Notification Service (Микросервис уведомлений)

**Версия:** 1.0  
**Дата:** 30.04.2026  
**Связь с архитектурой:** [`c4_Level_2_containers_diagram_Card2CardTransfer_v1.plantuml`](c4_Level_2_containers_diagram_Card2CardTransfer_v1.plantuml)  
**Связь с BRD:** [`card2card_transfer_brd.md`](card2card_transfer_brd.md)  
**Связь с Transfer Service:** [`transfer_service_backend.md`](transfer_service_backend.md)  

---

## 1. Общий обзор

### 1.1. Назначение
Notification Service — микросервис для отправки уведомлений пользователям о статусе переводов с карты на карту. Сервис отвечает за:
- Отправку push-уведомлений в мобильное приложение
- Отправку SMS-уведомлений
- Отправку email-уведомлений
- Управление шаблонами уведомлений
- Настройку предпочтений пользователя по каналам уведомлений
- Доставку уведомлений с гарантией (retry + dead letter queue)

### 1.2. Высокоуровневая логика
```
Получение события → Определение каналов → Генерация контента → 
Отправка → Фиксация статуса доставки
```

### 1.3. Технологический стек
- **Язык:** Java 17+
- **Фреймворк:** Spring Boot 3.x
- **БД:** PostgreSQL (Notification Database)
- **Очередь:** Kafka Consumer (подписка на события переводов)
- **Внешние интеграции:** Firebase Cloud Messaging (push), SMS-провайдер, SMTP/Email-провайдер
- **Кэш:** Redis (шаблоны уведомлений, предпочтения пользователей)

---

## 2. Входные данные

### 2.1. Kafka Consumer — События переводов

```yaml
Топик: banking.transfer.events
Группа: notification-service-group
Типы событий:
  - TransferInitiated       # Уведомление: "Перевод обрабатывается"
  - TransferCompleted       # Уведомление: "Перевод выполнен"
  - TransferFailed          # Уведомление: "Перевод не выполнен"
  - TransferCancelled       # Уведомление: "Перевод отменён"
```

### 2.2. Kafka Consumer — События лимитов

```yaml
Топик: banking.limit.events
Группа: notification-service-group
Типы событий:
  - LimitWarning            # Уведомление: "Использовано > 80% лимита"
  - LimitCritical           # Уведомление: "Использовано > 95% лимита"
  - LimitExhausted          # Уведомление: "Лимит исчерпан"
```

### 2.3. REST API — Управление предпочтениями

```typescript
// Получение предпочтений пользователя
GET /api/v1/notifications/preferences/{userId}
Response:
{
  "userId": "UUID",
  "channels": {
    "push": { "enabled": true, "deviceTokens": ["token1", "token2"] },
    "sms": { "enabled": true, "phone": "+79001234567" },
    "email": { "enabled": false, "email": "user@example.com" }
  },
  "events": {
    "transfer_completed": { "push": true, "sms": true, "email": false },
    "transfer_failed": { "push": true, "sms": true, "email": true },
    "limit_warning": { "push": true, "sms": false, "email": false }
  }
}

// Обновление предпочтений
PUT /api/v1/notifications/preferences/{userId}
Request Body:
{
  "channels": {
    "push": { "enabled": true, "deviceTokens": ["token1"] },
    "sms": { "enabled": false }
  },
  "events": {
    "transfer_completed": { "push": true, "sms": false }
  }
}
```

### 2.4. REST API — Регистрация устройства

```typescript
POST /api/v1/notifications/devices
Request Body:
{
  "userId": "UUID",
  "deviceToken": "fcm-token-here",
  "platform": "IOS | ANDROID",
  "deviceName": "iPhone 15 Pro"
}
Response:
{
  "registered": true,
  "deviceId": "UUID"
}

DELETE /api/v1/notifications/devices/{deviceId}
```

### 2.5. REST API — Административные

```typescript
// Получение статистики доставки
GET /api/v1/admin/notifications/stats?from=2026-04-01&to=2026-04-30
Response:
{
  "totalSent": 15000,
  "delivered": 14850,
  "failed": 150,
  "deliveryRate": 0.99,
  "byChannel": {
    "push": { "sent": 10000, "delivered": 9950 },
    "sms": { "sent": 3000, "delivered": 2950 },
    "email": { "sent": 2000, "delivered": 1950 }
  }
}

// Повторная отправка уведомления
POST /api/v1/admin/notifications/resend/{notificationId}
```

---

## 3. Валидации

### 3.1. Структурные валидации

| Поле | Проверка | Ошибка |
|------|----------|--------|
| `userId` | Валидный UUID | INVALID_USER_ID |
| `deviceToken` | Не пустой, длина ≤ 512 | INVALID_DEVICE_TOKEN |
| `platform` | Одно из: IOS, ANDROID | INVALID_PLATFORM |
| `phone` | Формат +7XXXXXXXXXX | INVALID_PHONE |
| `email` | Валидный email (RFC 5322) | INVALID_EMAIL |
| `eventType` | Одно из допустимых событий | INVALID_EVENT_TYPE |

### 3.2. Бизнес-валидации

**3.2.1. Проверка каналов:**
- Если канал отключён в предпочтениях → не отправлять
- Если нет deviceToken для push → пропустить канал
- Если нет phone для SMS → пропустить канал
- Если нет email для email → пропустить канал

**3.2.2. Проверка дублирования:**
- Одно и то же уведомление не отправляется дважды (idempotency key = eventId + userId)
- Если уведомление уже отправлено → пропустить

**3.2.3. Rate Limiting:**
- Не более 50 push-уведомлений в час на пользователя
- Не более 10 SMS в час на пользователя
- Не более 20 email в час на пользователя

### 3.3. Административные валидации

- Нельзя отключить все каналы уведомлений (минимум 1 канал)
- Нельзя отключить уведомления о failed переводах (обязательный канал)

---

## 4. Основная логика

### 4.1. Процесс обработки события (NotificationProcessor)

```
ШАГ 1: ПОЛУЧЕНИЕ СОБЫТИЯ ИЗ KAFKA
  ├── Десериализация события
  ├── Проверка idempotency key (eventId + userId)
  ├── Если уже обработано → SKIP
  └── Если новое → переход к шагу 2

ШАГ 2: ЗАГРУЗКА ПРЕДПОЧТЕНИЙ ПОЛЬЗОВАТЕЛЯ
  ├── Поиск в Redis (TTL: 10 минут)
  ├── Если не найдено → загрузка из PostgreSQL
  └── Определение активных каналов для данного типа события

ШАГ 3: ГЕНЕРАЦИЯ КОНТЕНТА
  ├── Загрузка шаблона уведомления (по типу события)
  ├── Подстановка переменных (сумма, статус, дата)
  ├── Локализация (язык пользователя)
  └── Генерация контента для каждого канала

ШАГ 4: ОТПРАВКА УВЕДОМЛЕНИЙ
  ├── Для каждого активного канала:
  │   ├── Push → Firebase Cloud Messaging
  │   ├── SMS → SMS-провайдер
  │   └── Email → SMTP-провайдер
  └── Фиксация статуса отправки

ШАГ 5: ФИКСАЦИЯ РЕЗУЛЬТАТА
  ├── INSERT INTO notification_log (userId, eventId, channels, status)
  ├── Если все каналы успешны → status = DELIVERED
  ├── Если частично успешны → status = PARTIALLY_DELIVERED
  └── Если все каналы неуспешны → status = FAILED → retry
```

### 4.2. Процесс генерации контента (ContentGenerator)

```
ШАГ 1: ЗАГРУЗКА ШАБЛОНА
  ├── Поиск шаблона по eventType + language
  ├── Если не найден → загрузка шаблона по умолчанию (RU)
  └── Если не найден → загрузка дефолтного шаблона

ШАГ 2: ПОДСТАНОВКА ПЕРЕМЕННЫХ
  ├── {amount} → сумма перевода (форматированная)
  ├── {currency} → валюта перевода
  ├── {status} → статус перевода
  ├── {date} → дата и время
  ├── {fromCard} → маскированный номер карты (****1234)
  ├── {toCard} → маскированный номер карты (****5678)
  └── {reason} → причина отказа (для failed)

ШАГ 3: ФОРМИРОВАНИЕ КОНТЕНТА ПО КАНАЛАМ
  ├── Push: заголовок (до 50 символов) + тело (до 200 символов)
  ├── SMS: только текст (до 160 символов)
  └── Email: subject (до 100 символов) + body (HTML)
```

### 4.3. Процесс отправки push-уведомлений (PushSender)

```
ШАГ 1: ПОЛУЧЕНИЕ DEVICE TOKEN
  ├── Загрузка всех deviceToken пользователя
  ├── Фильтрация активных токенов
  └── Если токенов нет → SKIP

ШАГ 2: ОТПРАВКА ЧЕРЕЗ FCM
  ├── POST https://fcm.googleapis.com/fcm/send
  ├── Payload: { notification: { title, body }, data: { eventId, type } }
  └── Обработка ответа

ШАГ 3: ОБРАБОТКА РЕЗУЛЬТАТА
  ├── Если success → статус DELIVERED
  ├── Если InvalidRegistration → удалить токен
  ├── Если NotRegistered → удалить токен
  └── Если Unavailable → retry с exponential backoff
```

### 4.4. Процесс отправки SMS (SmsSender)

```
ШАГ 1: ПРОВЕРКА НОМЕРА
  ├── Валидация формата (+7XXXXXXXXXX)
  └── Проверка на наличие в чёрном списке

ШАГ 2: ОТПРАВКА ЧЕРЕЗ SMS-ПРОВАЙДЕР
  ├── POST /api/v1/sms/send (SMS-провайдер)
  ├── Payload: { phone, text, sender: "BankName" }
  └── Обработка ответа

ШАГ 3: ОБРАБОТКА РЕЗУЛЬТАТА
  ├── Если success → статус DELIVERED
  ├── Если insufficient balance → алерт администратору
  └── Если timeout → retry с другим провайдером
```

### 4.5. Процесс отправки email (EmailSender)

```
ШАГ 1: ПРОВЕРКА АДРЕСА
  ├── Валидация формата email
  └── Проверка bounce-листа

ШАГ 2: ОТПРАВКА ЧЕРЕЗ SMTP
  ├── Формирование MIME-сообщения (HTML + plain text)
  ├── Отправка через SMTP-сервер
  └── Обработка ответа

ШАГ 3: ОБРАБОТКА РЕЗУЛЬТАТА
  ├── Если success → статус DELIVERED
  ├── Если bounce → добавить в bounce-лист
  └── Если timeout → retry
```

---

## 5. Интеграции

### 5.1. Внутренние сервисы

| Сервис | Протокол | Метод | Назначение |
|--------|----------|-------|------------|
| **Transfer Service** | Kafka | Consumer | Получение событий переводов |
| **Limit Service** | Kafka | Consumer | Получение событий лимитов |
| **Template Service** | REST | `GET /api/v1/templates/{eventType}` | Загрузка шаблонов уведомлений |
| **Admin Panel** | REST | `GET /api/v1/admin/notifications/*` | Статистика и управление |

### 5.2. Внешние системы

| Система | Протокол | Метод | Назначение |
|---------|----------|-------|------------|
| **Firebase Cloud Messaging** | HTTP | POST /fcm/send | Отправка push-уведомлений |
| **SMS-провайдер** | REST | POST /api/v1/sms/send | Отправка SMS |
| **SMTP-сервер** | SMTP | MIME | Отправка email |

### 5.3. Kafka

**Топик:** `banking.transfer.events` (Consumer)

```yaml
Группа: notification-service-group
Client ID: notification-service
Auto Commit: false
Max Poll Records: 50
```

**Топик:** `banking.limit.events` (Consumer)

```yaml
Группа: notification-service-group
Client ID: notification-service
Auto Commit: false
Max Poll Records: 50
```

### 5.4. База данных

```sql
-- Таблица предпочтений пользователей
CREATE TABLE notification_preferences (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL UNIQUE,
    push_enabled BOOLEAN DEFAULT true,
    sms_enabled BOOLEAN DEFAULT true,
    email_enabled BOOLEAN DEFAULT false,
    phone VARCHAR(20),
    email VARCHAR(255),
    event_preferences JSONB,          -- { "transfer_completed": { "push": true, "sms": true } }
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица устройств для push-уведомлений
CREATE TABLE user_devices (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    device_token VARCHAR(512) NOT NULL,
    platform VARCHAR(10) NOT NULL,     -- IOS, ANDROID
    device_name VARCHAR(255),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, device_token)
);

-- Таблица логов уведомлений
CREATE TABLE notification_log (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    event_id UUID NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    channels JSONB NOT NULL,           -- { "push": "DELIVERED", "sms": "FAILED" }
    status VARCHAR(20) NOT NULL,       -- DELIVERED, PARTIALLY_DELIVERED, FAILED, PENDING
    error_details JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    delivered_at TIMESTAMP
);

-- Таблица шаблонов уведомлений (кэш)
CREATE TABLE notification_templates (
    id UUID PRIMARY KEY,
    event_type VARCHAR(50) NOT NULL,
    language VARCHAR(2) NOT NULL DEFAULT 'RU',
    channel VARCHAR(10) NOT NULL,      -- PUSH, SMS, EMAIL
    title VARCHAR(100),
    body TEXT NOT NULL,
    variables JSONB,                   -- ["amount", "currency", "status"]
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(event_type, language, channel)
);

-- Таблица bounce-листа (невалидные адреса)
CREATE TABLE bounce_list (
    id UUID PRIMARY KEY,
    email VARCHAR(255),
    phone VARCHAR(20),
    reason VARCHAR(100),
    bounced_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Индексы
CREATE INDEX idx_notification_preferences_user_id ON notification_preferences(user_id);
CREATE INDEX idx_user_devices_user_id ON user_devices(user_id);
CREATE INDEX idx_user_devices_token ON user_devices(device_token);
CREATE INDEX

CREATE INDEX idx_notification_log_event_id ON notification_log(event_id);
CREATE INDEX idx_notification_log_user_id ON notification_log(user_id);
CREATE INDEX idx_notification_log_status ON notification_log(status);
CREATE INDEX idx_notification_templates_event_type ON notification_templates(event_type);
CREATE INDEX idx_bounce_list_email ON bounce_list(email);

---

## 6. Исключительные ситуации

### 6.1. Валидационные ошибки (HTTP 400)

| Код ошибки | Сообщение | Условие |
|------------|-----------|---------|
| `INVALID_USER_ID` | "Некорректный идентификатор пользователя" | userId не UUID |
| `INVALID_DEVICE_TOKEN` | "Некорректный токен устройства" | deviceToken пустой или > 512 символов |
| `INVALID_PLATFORM` | "Неподдерживаемая платформа" | platform не IOS/ANDROID |
| `INVALID_PHONE` | "Некорректный номер телефона" | phone не соответствует формату |
| `INVALID_EMAIL` | "Некорректный email адрес" | email не проходит RFC 5322 |
| `INVALID_EVENT_TYPE` | "Некорректный тип события" | eventType не из допустимого списка |

### 6.2. Бизнес-ошибки (HTTP 422)

| Код ошибки | Сообщение | Условие |
|------------|-----------|---------|
| `ALL_CHANNELS_DISABLED` | "Все каналы уведомлений отключены" | Минимум 1 канал должен быть включён |
| `MANDATORY_CHANNEL_DISABLED` | "Нельзя отключить уведомления о failed переводах" | Попытка отключить обязательный канал |
| `RATE_LIMIT_EXCEEDED` | "Превышен лимит уведомлений" | > 50 push / > 10 SMS / > 20 email в час |
| `DUPLICATE_NOTIFICATION` | "Уведомление уже отправлено" | idempotency key уже существует |
| `NO_ACTIVE_DEVICES` | "Нет активных устройств для push" | Все deviceToken неактивны |
| `DEVICE_NOT_FOUND` | "Устройство не найдено" | deviceId не существует |

### 6.3. Интеграционные ошибки (HTTP 502/503)

| Код ошибки | Сообщение | Условие | Действие |
|------------|-----------|---------|----------|
| `FCM_UNAVAILABLE` | "Firebase Cloud Messaging недоступен" | FCM не отвечает > 3 сек | Retry 3 раза, затем DLQ |
| `FCM_AUTH_ERROR` | "Ошибка аутентификации FCM" | Невалидный API key | Алерт администратору |
| `SMS_PROVIDER_ERROR` | "SMS-провайдер недоступен" | SMS-провайдер не отвечает | Retry с другим провайдером |
| `SMS_INSUFFICIENT_BALANCE` | "Недостаточно средств на счете SMS" | Баланс SMS-провайдера < порога | Алерт администратору |
| `SMTP_UNAVAILABLE` | "SMTP-сервер недоступен" | SMTP не отвечает > 5 сек | Retry 3 раза, затем DLQ |
| `SMTP_AUTH_ERROR` | "Ошибка аутентификации SMTP" | Невалидные учетные данные | Алерт администратору |
| `TEMPLATE_NOT_FOUND` | "Шаблон уведомления не найден" | Нет шаблона для eventType + language | Использовать дефолтный шаблон |
| `KAFKA_CONSUMER_ERROR` | "Ошибка обработки события Kafka" | Ошибка десериализации | Отправка в DLQ |

### 6.4. Стратегии восстановления

**6.4.1. Retry политики:**

| Канал | Max retries | Backoff | DLQ |
|-------|-------------|---------|-----|
| Push (FCM) | 3 | 1s, 5s, 30s | notification.push.dlq |
| SMS | 3 | 1s, 10s, 60s | notification.sms.dlq |
| Email (SMTP) | 3 | 5s, 30s, 120s | notification.email.dlq |

**6.4.2. Graceful degradation:**
- При недоступности FCM → отправить только SMS и email
- При недоступности SMS → отправить push и email
- При недоступности всех каналов → сохранить в БД для повторной отправки
- При недоступности Template Service → использовать шаблоны из локального кэша

**6.4.3. Dead Letter Queue обработка:**
- Уведомления в DLQ проверяются раз в 15 минут
- После 3 неудачных попыток из DLQ → уведомление помечается как LOST
- Администратор может вручную инициировать повторную отправку

---

## 7. Выходные данные

### 7.1. REST API — Успешные ответы

**7.1.1. Предпочтения пользователя (200):**
```json
{
  "userId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "channels": {
    "push": { "enabled": true, "deviceTokens": ["token1", "token2"] },
    "sms": { "enabled": true, "phone": "+79001234567" },
    "email": { "enabled": false }
  },
  "events": {
    "transfer_completed": { "push": true, "sms": true, "email": false },
    "transfer_failed": { "push": true, "sms": true, "email": true },
    "limit_warning": { "push": true, "sms": false, "email": false }
  },
  "updatedAt": "2026-04-30T14:30:00Z"
}
```

**7.1.2. Регистрация устройства (201):**
```json
{
  "registered": true,
  "deviceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "registeredAt": "2026-04-30T14:30:00Z"
}
```

**7.1.3. Статистика доставки (200):**
```json
{
  "period": { "from": "2026-04-01", "to": "2026-04-30" },
  "totalSent": 15000,
  "delivered": 14850,
  "failed": 150,
  "deliveryRate": 0.99,
  "byChannel": {
    "push": { "sent": 10000, "delivered": 9950, "deliveryRate": 0.995 },
    "sms": { "sent": 3000, "delivered": 2950, "deliveryRate": 0.983 },
    "email": { "sent": 2000, "delivered": 1950, "deliveryRate": 0.975 }
  },
  "byEventType": {
    "transfer_completed": { "sent": 8000, "delivered": 7950 },
    "transfer_failed": { "sent": 500, "delivered": 500 },
    "limit_warning": { "sent": 6500, "delivered": 6400 }
  }
}
```

### 7.2. Kafka события (Producer)

**Топик:** `banking.notification.events`

| Событие | Условие | Payload |
|---------|---------|---------|
| `NotificationSent` | Уведомление отправлено | `{ userId, eventId, channels, status }` |
| `NotificationFailed` | Уведомление не доставлено | `{ userId, eventId, channels, error }` |
| `DeviceRegistered` | Новое устройство | `{ userId, deviceId, platform }` |
| `DeviceRemoved` | Устройство удалено | `{ userId, deviceId, reason }` |

---

## 8. Производительность

### 8.1. Целевые показатели

| Метрика | Целевое значение | Метод измерения |
|---------|-----------------|-----------------|
| Время обработки события | ≤ 500 мс (p99) | JMeter, Gatling |
| Время отправки push | ≤ 200 мс (p99) | Мониторинг FCM |
| Время отправки SMS | ≤ 2 сек (p99) | Мониторинг SMS-провайдера |
| Время отправки email | ≤ 5 сек (p99) | Мониторинг SMTP |
| Пропускная способность | 1000 событий/сек | Load testing |
| Доставка push | ≥ 99% | Статистика FCM |
| Доставка SMS | ≥ 98% | Статистика SMS-провайдера |
| Доставка email | ≥ 97% | Статистика SMTP |

### 8.2. Оптимизации

**8.2.1. Кэширование:**
- **Redis Cache:** предпочтения пользователя (TTL: 10 минут)
- **Redis Cache:** шаблоны уведомлений (TTL: 1 час)
- **Local Cache (Caffeine):** активные deviceToken (TTL: 5 минут)

**8.2.2. Пакетная обработка:**
- Kafka consumer: max.poll.records = 50
- Batch insert для notification_log: до 100 записей за транзакцию
- Пул соединений с FCM: до 10 параллельных запросов

**8.2.3. Асинхронная обработка:**
- Отправка по каждому каналу в отдельном потоке
- CompletableFuture для параллельной отправки push + SMS + email
- Timeout на каждый канал: push 3s, SMS 5s, email 10s

### 8.3. Ограничения

| Параметр | Значение | Обоснование |
|----------|----------|-------------|
| Максимальная нагрузка | 2000 событий/сек | Пиковые нагрузки (Черная пятница) |
| Максимальное время обработки | 10 сек | Таймаут для всего пайплайна |
| Максимальный размер push | 4 KB | Ограничение FCM |
| Максимальная длина SMS | 160 символов | Стандарт GSM |
| Максимальный размер email | 10 MB | Ограничение SMTP |
| Максимум устройств на пользователя | 10 | Защита от спама |

---

## 9. Безопасность

### 9.1. Аутентификация и авторизация

| Endpoint | Метод | Роль | Аутентификация |
|----------|-------|------|----------------|
| `GET /api/v1/notifications/preferences/{userId}` | Внутренний | Любой сервис | Service-to-Service JWT |
| `PUT /api/v1/notifications/preferences/{userId}` | Пользовательский | USER | JWT + userId match |
| `POST /api/v1/notifications/devices` | Пользовательский | USER | JWT + userId match |
| `DELETE /api/v1/notifications/devices/{deviceId}` | Пользовательский | USER | JWT + владелец устройства |
| `GET /api/v1/admin/notifications/*` | Административный | ADMIN | JWT + Role check |

### 9.2. Защита данных

- **PCI DSS:** уведомления не содержат полные номера карт (только маскированные: ****1234)
- **Шифрование:** deviceToken хранятся в зашифрованном виде (AES-256)
- **Маскирование:** номера телефонов маскируются в логах (+7***1234)
- **Логирование:** без записи персональных данных (PII) в логи
- **Bounce-лист:** автоматическое удаление невалидных адресов

### 9.3. Rate Limiting

| Endpoint | Лимит | Период |
|----------|-------|--------|
| `PUT /api/v1/notifications/preferences/*` | 10 запросов | 1 минута |
| `POST /api/v1/notifications/devices` | 5 запросов | 1 минута |
| `DELETE /api/v1/notifications/devices/*` | 5 запросов | 1 минута |
| `GET /api/v1/admin/notifications/*` | 30 запросов | 1 минута |

### 9.4. Аудит

Все операции логируются:
- Изменение предпочтений (кто, когда, какие каналы)
- Регистрация/удаление устройств
- Статусы доставки уведомлений
- Ошибки интеграций с внешними системами
- Повторные отправки из DLQ