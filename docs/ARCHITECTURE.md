
# 📐 Техническая документация TaxiService

## Оглавление

1. [Обзор архитектуры](#1-обзор-архитектуры)
2. [Микросервисы](#2-микросервисы)
3. [Базы данных](#3-базы-данных)
4. [Межсервисное взаимодействие](#4-межсервисное-взаимодействие)
5. [Real-time коммуникация](#5-real-time-коммуникация)
6. [Аутентификация и авторизация](#6-аутентификация-и-авторизация)
7. [Event-Driven Architecture](#7-event-driven-architecture)
8. [Кеширование](#8-кеширование)
9. [Деплой и инфраструктура](#9-деплой-и-инфраструктура)

---

## 1. Обзор архитектуры

### 1.1 Принципы проектирования

- Микросервисная архитектура — каждый сервис отвечает за свой bounded context
- Database per Service — изолированные базы данных для каждого сервиса
- Event-Driven — асинхронное взаимодействие через RabbitMQ
- CQRS-lite — разделение записи и чтения где это оправдано
- API Gateway — единая точка входа для клиентов

### 1.2 Высокоуровневая схема
```
                                    ┌──────────────┐
                                    │   Clients    │
                                    │ (SolidJS)    │
                                    └──────┬───────┘
                                           │ HTTPS
                                           ▼
                                    ┌──────────────┐
                                    │ API Gateway  │
                                    │   (Kong)     │
                                    └──────┬───────┘
                                           │
         ┌─────────────┬───────────────────┼───────────────────┬─────────────┐
         ▼             ▼                   ▼                   ▼             ▼  
   ┌───────────┐ ┌───────────┐      ┌───────────┐      ┌───────────┐ ┌───────────┐
   │   Auth    │ │   User    │      │   Ride    │      │ Location  │ │  Admin    │
   │  Service  │ │  Service  │      │  Service  │      │  Service  │ │  Service  │
   │   (C#)    │ │   (C#)    │      │   (Go)    │      │   (Go)    │ │   (C#)    │
   └─────┬─────┘ └─────┬─────┘      └─────┬─────┘      └─────┬─────┘ └─────┬─────┘
         │             │                   │                   │             │
         │             │            ┌──────┴──────┐            │             │
         │             │            ▼             ▼            │             │
         │             │      ┌───────────┐ ┌───────────┐      │             │
         │             │      │ Matching  │ │  Pricing  │      │             │
         │             │      │  Service  │ │  Service  │      │             │
         │             │      │   (Go)    │ │   (Go)    │      │             │
         │             │      └─────┬─────┘ └─────┬─────┘      │             │
         │             │            │             │            │             │
         └─────────────┴────────────┴──────┬──────┴────────────┴─────────────┘
                                           │
                              ┌────────────┴────────────┐
                              ▼                         ▼
                       ┌─────────────┐          ┌─────────────┐
                       │  RabbitMQ   │          │    Redis    │
                       │  (Events)   │          │(Cache/Pub)  │
                       └─────────────┘          └─────────────┘
                              │
         ┌────────────────────┴────────────────────┐
         ▼                                         ▼
   ┌────────────┐                           ┌───────────┐
   │Notification│                           │ Analytics │
   │  Service   │                           │ (в Admin) │
   │   (C#)     │                           └───────────┘
   └────────────┘
```

### 1.3 Распределение по технологиям

| Язык | Сервисы | Причина выбора |
|------|---------|----------------|
| Go | Ride, Matching, Location, Pricing | Высокая нагрузка, real-time, concurrency |
| C# .NET | Auth, User, Notification, Admin | CRUD операции, бизнес-логика, интеграции |

---

## 2. Микросервисы

### 2.1 API Gateway

Ответственность: Единая точка входа, маршрутизация, безопасность

Функции:
- SSL termination
- Rate limiting (по IP, по user_id)
- JWT валидация
- Request routing
- Load balancing
- Request/Response logging
- Trace initialization

Конфигурация rate limits:  
  

Пассажиры:
  - Создание поездки: 5 req/min
  - Отмена: 3 req/min
  - Прочие запросы: 60 req/min

Водители:
  - Обновление локации: 20 req/sec
  - Прочие запросы: 60 req/min

Общие:
  - Регистрация: 10 req/hour per IP
  - SMS код: 3 req/hour per phone
---

### 2.2 AuthService (C# .NET)

Ответственность: Аутентификация, авторизация, сессии

Порт: 8085

Endpoints:  
POST   /api/auth/register        # Регистрация  
POST   /api/auth/login           # Авторизация  
POST   /api/auth/refresh-token   # Обновление токена  
POST   /api/auth/logout          # Выход  
POST   /api/auth/verify-phone    # Верификация телефона  
POST   /api/auth/resend-code     # Повторная отправка кода  
GET    /api/auth/health          # Health check  
База данных: auth_db (PostgreSQL)

Таблицы:
- users — базовая информация пользователей
- sessions — активные сессии и refresh tokens

Redis ключи:  
refresh_token:{token}     → user_id (TTL: 30 days)  
blacklist:{token}         → 1 (TTL: до expiry токена)  
verification:{phone}      → code (TTL: 5 min)  
  
События (публикует):
- user.registered
- user.logged_in
- user.blocked

---

### 2.3 UserService (C# .NET)

Ответственность: Профили пользователей, рейтинги, автомобили

Порт: 8086

Endpoints:  
GET    /api/users/:id              # Получить пользователя  
PUT    /api/users/:id              # Обновить пользователя  
GET    /api/users/:id/profile      # Профиль (passenger/driver)  
PUT    /api/users/:id/profile      # Обновить профиль  
DELETE /api/users/:id              # Удалить пользователя  

# Рейтинги (встроено)
POST   /api/ratings                # Создать оценку  
GET    /api/ratings/user/:user_id  # Оценки пользователя  
GET    /api/ratings/stats/:user_id # Статистика рейтинга  
База данных: user_db (PostgreSQL)

Таблицы:
- passenger_profiles — профили пассажиров
- driver_profiles — профили водителей
- cars — автомобили водителей
- ratings — оценки поездок
- rating_stats — агрегированная статистика

Redis ключи:
user:profile:{user_id}    → JSON (TTL: 1 hour)  
driver:online:{user_id}   → boolean  
  
События:
- Слушает: user.registered, ride.completed
- Публикует: profile.updated, driver.went_online, driver.went_offline

---

### 2.4 RideService (Go)

Ответственность: Жизненный цикл поездок, платежи (MVP)

Порт: 8081

Endpoints:  
POST   /api/rides/create           # Создать поездку  
GET    /api/rides/:id              # Получить поездку  
PUT    /api/rides/:id/accept       # Принять (водитель)  
PUT    /api/rides/:id/start        # Начать поездку  
PUT    /api/rides/:id/complete     # Завершить поездку  
PUT    /api/rides/:id/cancel       # Отменить  
GET    /api/rides/active           # Активная поездка  
GET    /api/rides/history          # История поездок  
  
gRPC (внутренний):  
service RideService {  
  rpc CreateRide(CreateRideRequest) returns (RideResponse);  
  rpc GetRide(GetRideRequest) returns (RideResponse);  
  rpc UpdateRideStatus(UpdateStatusRequest) returns (RideResponse);  
}  
База данных: ride_db (PostgreSQL)

Таблицы:
- rides — поездки
- payments — платежи (MVP: только наличные)

Redis ключи:  
passenger:active_ride:{id}  → ride_id (TTL: 2h)  
driver:active_ride:{id}     → ride_id (TTL: 2h)  
ride:active:{ride_id}       → JSON (TTL: 2h)  
ride:requests:{passenger_id}→ count (TTL: 1 min) # rate limit 
   
События:
- Слушает: matching.driver_assigned, payment.completed
- Публикует: ride.created, ride.accepted, ride.started, ride.completed, ride.cancelled

---

### 2.5 MatchingService (Go)

Ответственность: Поиск и назначение водителей

Порт: 8082

Endpoints:  
POST   /api/matching/find-driver      # Начать поиск  
POST   /api/matching/driver-response  # Ответ водителя  
GET    /api/matching/status/:ride_id  # Статус поиска  
  
WebSocket:  
/ws/matching/:ride_id           # Для пассажира (статус)  
/ws/matching/driver/:driver_id  # Для водителя (новые заказы)  
  
Алгоритм матчинга:  
1. Получение события ride.created
2. Поиск водителей через Redis GEORADIUS (5км → 10км → 15км)
3. Фильтрация по категории и статусу
4. Уведомление водителей через Redis Pub/Sub
5. Ожидание ответа (30 сек таймаут)
6. При принятии — публикация matching.driver_assigned

Redis ключи:
matching:ride:{ride_id}                    → {status, attempts, notified_drivers[]}  
matching:driver_timeout:{driver}:{ride}   → 1 (TTL: 30s)  
  
События:
- Слушает: ride.created, driver.went_online, driver.went_offline
- Публикует: matching.searching, matching.driver_assigned, matching.failed

---

### 2.6 LocationService (Go)

Ответственность: Геолокация, трекинг в реальном времени

Порт: 8083

Endpoints:  
POST   /api/location/update           # Обновить локацию  
GET    /api/location/driver/:id       # Локация водителя  
GET    /api/location/nearby           # Ближайшие водители  
GET    /api/location/eta              # Расчёт ETA  
  
WebSocket:  
/ws/location/driver/:driver_id  # Водитель отправляет локацию  
/ws/location/track/:ride_id     # Пассажир получает трек  
  
Redis ключи:  
drivers:geo:online              → GEOADD (водители онлайн)  
driver:location:{driver_id}     → {lat, lng, heading, speed} (TTL: 30s)  
ride:track:{ride_id}            → LIST of locations (LTRIM 1000)  
  
Redis Pub/Sub каналы:  
location:updates:{ride_id}      # Обновления для пассажира  
  
MongoDB коллекция: location_history  
{  
  ride_id: "uuid",  
  driver_id: "uuid",   
  track: [{lat, lng, timestamp, speed}, ...],  
  created_at: ISODate,  
  expire_at: ISODate  // TTL 90 days  
}  
  
События:  
- Слушает: ride.accepted, ride.started, ride.completed  
- Публикует: location.updated, driver.arrived  

---

### 2.7 PricingService (Go)

Ответственность: Расчёт стоимости, тарифы

Порт: 8084

Endpoints:  
POST   /api/pricing/estimate          # Предварительная стоимость  
POST   /api/pricing/calculate-final   # Финальная стоимость  
GET    /api/pricing/rules             # Получить тарифы  
PUT    /api/pricing/rules/:id         # Обновить тариф (admin)  
  
gRPC (внутренний):  
service PricingService {  
  rpc CalculatePrice(PriceRequest) returns (PriceResponse);  
}  
  
Формула расчёта:  
price = base_price + (distance_km × price_per_km) + (duration_min × price_per_minute)  
price = max(price, minimum_price)  
price = price × surge_multiplier  // После MVP  
price = price - discount          // После MVP  
База данных: pricing_db (PostgreSQL)  

Таблицы:
- pricing_rules — тарифы по категориям
- surge_zones — зоны повышенного спроса (после MVP)
- promo_codes — промокоды (после MVP)

Redis ключи:
pricing:rules:{category}:{city}  → JSON (TTL: 1 hour)  
pricing:surge:{zone_id}          → multiplier (TTL: 5 min)

---

### 2.8 NotificationService (C# .NET)

Ответственность: Уведомления (push, sms, email)

Порт: 8087

Endpoints:  
POST   /api/notifications/send           # Отправить уведомление  
POST   /api/notifications/device-token   # Регистрация устройства  
GET    /api/notifications/user/:id       # Уведомления пользователя  
PUT    /api/notifications/preferences    # Настройки уведомлений  
База данных: notification_db (PostgreSQL)

Таблицы:
- notifications — история уведомлений
- device_tokens — токены устройств (FCM/APNs)
- notification_preferences — настройки пользователей

RabbitMQ очереди:  
notifications.priority   # Срочные (ride updates)  
notifications.standard   # Обычные  
notifications.email.bulk # Email рассылки  
  
Интеграции (после MVP):
- Firebase Cloud Messaging (push)
- Twilio (SMS)
- SendGrid (email)

События:
- Слушает: ВСЕ события системы (ride.*, matching.*, payment.*)

---

### 2.9 AdminService (C# .NET)

Ответственность: Административные функции, аналитика

Порт: 8090

Endpoints:
# Пользователи
GET    /api/admin/users              # Список пользователей  
GET    /api/admin/users/:id          # Детали пользователя  
PUT    /api/admin/users/:id/block    # Заблокировать  
PUT    /api/admin/users/:id/unblock  # Разблокировать

# Поездки
GET    /api/admin/rides              # Список поездок  
GET    /api/admin/rides/:id          # Детали поездки  
PUT    /api/admin/rides/:id/cancel   # Отменить поездку

# Тарифы
GET    /api/admin/pricing/rules      # Тарифы  
POST   /api/admin/pricing/rules      # Создать тариф  
PUT    /api/admin/pricing/rules/:id  # Обновить тариф

# Аналитика
GET    /api/admin/analytics/overview    # Общий дашборд  
GET    /api/admin/analytics/rides       # Статистика поездок  
GET    /api/admin/analytics/revenue     # Финансы  
GET    /api/admin/analytics/drivers     # Статистика водителей  
База данных: Использует базы других сервисов (read-only) + admin_db для логов

---

## 3. Базы данных

### 3.1 PostgreSQL (Primary Storage)

Отдельные базы для каждого сервиса:

```
┌─────────────────────────────────────────────────────────────────┐
│                       PostgreSQL Cluster                        │
├─────────────────────────────────────────────────────────────────┤
│ auth_db      │ user_db              │ ride_db      │ pricing_db │
│ - users      │ - passenger_profiles │ - rides      │ - pricing_ │
│ - sessions   │                      │ - payments   │   rules    │
│              │ - driver_profiles    │              │ - surge_   │
│              │                      │              │   zones    │
│              │ - cars               │              │ - promo_   │
│              │ - ratings            │              │   codes    │
│              │ - rating_            │              │            │
│              │   stats              │              │            │
├─────────────────────────────────────────────────────────────────┤
│ notification_db              │ admin_db                         │
│ - notifications              │ - audit_logs                     │
│ - device_tokens              │ - admin_actions                  │
│ - notification_preferences   │                                  │
└─────────────────────────────────────────────────────────────────┘
```
### 3.2 Redis (Cache & Real-time)

Используется для:
- Кеширование (профили, тарифы)
- Геопространственные индексы (GEOADD/GEORADIUS)
- Pub/Sub (real-time updates)
- Сессии и токены
- Rate limiting
- Distributed locks

### 3.3 MongoDB (Logs & History)

Коллекции:
- location_history — треки поездок (TTL 90 дней)
- service_logs — логи сервисов (TTL 30 дней)
- audit_logs — аудит действий (permanent)

---

## 4. Межсервисное взаимодействие

### 4.1 Синхронное (REST/gRPC)

Клиент → API Gateway → Сервис

Ride Service → Pricing Service (gRPC)  
Ride Service → User Service (REST - проверка профиля)  
Admin Service → All Services (REST - read-only)
### 4.2 Асинхронное (RabbitMQ)

Producer → Exchange → Queue → Consumer

Exchanges:
- ride.events (topic)
- matching.events (topic)  
- payment.events (topic)
- user.events (topic)
- notification.events (fanout)

### 4.3 Real-time (WebSocket + Redis Pub/Sub)

```
Driver App ←→ Location Service (WebSocket)
              ↓
           Redis Pub/Sub
              ↓
Passenger App ←→ Location Service (WebSocket)
```
---

## 5. Real-time коммуникация

### 5.1 WebSocket Endpoints

| Endpoint | Направление | Описание |
|----------|-------------|----------|
| /ws/location/driver/:id | Driver → Server | Отправка локации |
| /ws/location/track/:ride_id | Server → Passenger | Трекинг водителя |
| /ws/matching/driver/:id | Server → Driver | Новые заказы |
| /ws/matching/:ride_id | Server → Passenger | Статус поиска |

### 5.2 Протокол сообщений

```typescript
// Client → Server (Driver location)
{
  "type": "location_update",
  "data": {
    "lat": 55.7558,
    "lng": 37.6173,
    "heading": 180,
    "speed": 45,
    "timestamp": "2025-01-15T10:30:00Z"
  }
}

// Server → Client (Ride offer)
{
  "type": "new_ride_offer",
  "data": {
    "ride_id": "uuid",
    "passenger_name": "John",
    "passenger_rating": 4.8,
    "pickup": {"lat": 55.75, "lng": 37.61, "address": "..."},
    "dropoff": {"lat": 55.76, "lng": 37.62, "address": "..."},
    "estimated_price": 350,
    "expires_in": 30
  }
}

// Server → Client (Location update)
{
  "type": "driver_location",
  "data": {
    "lat": 55.7558,
    "lng": 37.6173,
    "eta_seconds": 180,
    "distance_meters": 1200
  }
}
```
---

## 6. Аутентификация и авторизация

### 6.1 JWT Tokens

**Access Token (1 час):**
```json
{
  "sub": "user-uuid",
  "role": "passenger|driver|admin",
  "iat": 1700000000,
  "exp": 1700003600,
  "jti": "token-id"
}
```

**Refresh Token (30 дней):**
- Хранится в Redis
- Ротация при каждом использовании
- Blacklist при logout

### 6.2 Роли и права

| Роль | Права |
|------|-------|
| `passenger` | Заказ поездок, просмотр своей истории, оценки |
| `driver` | Принятие заказов, обновление локации, просмотр своей статистики |
| `admin` | Полный доступ к админке, управление пользователями и тарифами |

---

## 7. Event-Driven Architecture

### 7.1 События системы

**Ride Events:**
ride.created    → Matching, Notification, Analytics  
ride.accepted   → Notification, Location, Analytics  
ride.started    → Notification, Location, Analytics  
ride.completed  → Payment, Notification, User (rating), Analytics  
ride.cancelled  → Notification, Analytics

**Matching Events:**
matching.searching       → Notification (пассажир)  
matching.driver_assigned → Ride, Notification, Location  
matching.failed          → Ride, Notification

**User Events:**
user.registered     → User (создание профиля)  
driver.went_online  → Matching  
driver.went_offline → Matching

### 7.2 Формат события

```json
{
  "event_id": "uuid",
  "event_type": "ride.created",
  "timestamp": "2025-01-15T10:30:00Z",
  "version": "1.0",
  "data": {
    "ride_id": "uuid",
    "passenger_id": "uuid",
    "pickup_location": {"lat": 55.75, "lng": 37.61},
    "dropoff_location": {"lat": 55.76, "lng": 37.62},
    "category": "economy",
    "estimated_price": 350
  },
  "metadata": {
    "correlation_id": "uuid",
    "causation_id": "uuid",
    "user_id": "uuid"
  }
}
```

---

## 8. Кеширование

### 8.1 Стратегия кеширования

| Данные | TTL | Invalidation |
|--------|-----|--------------|
| User profiles | 1 час | При обновлении профиля |
| Pricing rules | 1 час | При обновлении тарифов |
| Driver locations | 30 сек | Автоматически (TTL) |
| Active rides | 2 часа | При изменении статуса |
| JWT blacklist | До expiry токена | Никогда |

### 8.2 Cache-Aside Pattern

1. Проверить Redis
2. Если есть — вернуть
3. Если нет — запросить из БД
4. Сохранить в Redis с TTL
5. Вернуть результат

---

## 9. Деплой и инфраструктура

### 9.1 Docker Compose (Development)

```yaml
services:
  postgres:
    image: postgres:15
    ports: ["5432:5432"]
    
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    
  rabbitmq:
    image: rabbitmq:3-management
    ports: ["5672:5672", "15672:15672"]
    
  mongodb:
    image: mongo:6
    ports: ["27017:27017"]
    
  kong:
    image: kong:3.4
    ports: ["8000:8000", "8001:8001"]

  # ... services

```
### 9.2 Health Checks

Каждый сервис предоставляет:
GET /health → {"status": "healthy", "checks": {...}}

Проверяется:
- Database connection
- Redis connection
- RabbitMQ connection (если используется)

### 9.3 Логирование

**Формат (structured JSON):**

```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "level": "info",
  "service": "ride-service",
  "trace_id": "uuid",
  "message": "Ride created",
  "context": {
    "ride_id": "uuid",
    "user_id": "uuid"
  }
}
```

### 9.4 Metrics: после MVP

- **Metrics:** Tracing:+ Grafana
- **Tracing:** JLogs:penTelemetry
- **Logs:** ELK Stack (Elastick Stack)
- **Alerting:** Grafana Alerts

---

## Приложения

### A. Порты сервисов

| Сервис | Порт |
|--------|------|
| API Gateway | 8000 |
| Auth Service | 8085 |
| User Service | 8086 |
| Ride Service | 8081 |
| Matching Service | 8082 |
| Location Service | 8083 |
| Pricing Service | 8084 |
| Notification Service | 8087 |
| Admin Service | 8090 |

### B. Переменные окружения

См. .env.example в корне проекта.

### C. Полезные команды

```shell
# Запуск всего
docker-compose up -d

# Логи сервиса
docker-compose logs -f ride-service

# Миграции
./scripts/migrate.sh

# Тесты
./scripts/test.sh

# Линтинг
./scripts/lint.sh
```
