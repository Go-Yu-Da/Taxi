# 🚕 TaxiService

Микросервисная платформа для сервиса такси. Включает приложения для пассажиров, водителей и администраторов.

## 📋 Оглавление

- [О проекте](#о-проекте)
- [Возможности](#возможности)
- [Технологический стек](#технологический-стек)
- [Архитектура](#архитектура)
- [Быстрый старт](#быстрый-старт)
- [Структура проекта](#структура-проекта)
- [API документация](#api-документация)
- [Команда](#команда)
- [Лицензия](#лицензия)

## 🎯 О проекте

TaxiService — это полнофункциональная платформа для заказа такси, разработанная с использованием микросервисной архитектуры. Проект включает:

- Passenger App — мобильное/веб приложение для пассажиров
- Driver App — приложение для водителей
- Admin Panel — панель администратора

### Статус проекта

🚧 В активной разработке — MVP версия

## ✨ Возможности

### Для пассажиров
- 📱 Регистрация и авторизация по телефону
- 🗺️ Заказ такси с выбором точек на карте
- 💰 Предварительный расчёт стоимости поездки
- 📍 Отслеживание водителя в реальном времени
- 📜 История поездок
- ⭐ Оценка водителей

### Для водителей
- 🔔 Получение заказов в реальном времени
- ✅ Принятие/отклонение заказов
- 🧭 Навигация к пассажиру и точке назначения
- 📊 Статистика заработка
- 🟢 Управление статусом (онлайн/оффлайн)

### Для администраторов
- 👥 Управление пользователями
- 🚗 Мониторинг активных поездок
- 💵 Настройка тарифов
- 📈 Аналитика и отчёты

## 🛠 Технологический стек

### Backend
| Сервис | Технология | Описание |
|--------|------------|----------|
| API Gateway | Go 1.25 | Роутинг, rate limiting, SSL |
| AuthService | C# .NET 9 | Аутентификация, JWT |
| UserService | C# .NET 9 | Профили, рейтинги |
| RideService | Go 1.25 | Управление поездками |
| MatchingService | Go 1.25 | Подбор водителей |
| LocationService | Go 1.25 | Геолокация, трекинг |
| PricingService | Go 1.25 | Расчёт стоимости |
| NotificationService | C# .NET 9 | Push, SMS, Email |
| AdminService | C# .NET 9 | Админ панель API |

### Frontend
- Framework: SolidJS
- Styling: Tailwind CSS
- Maps: Google Maps API
- State: SolidJS Signals

### Infrastructure
- Database: PostgreSQL 17
- Cache: Redis 8.0
- Message Broker: RabbitMQ
- Logs Storage: MongoDB
- Containerization: Docker & Docker Compose
- API Gateway: Go

## 🏗 Архитектура

```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENTS                              │
│   Passenger App    │    Driver App    │    Admin Panel      │
│     (SolidJS)      │    (SolidJS)     │     (SolidJS)       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     API GATEWAY (Kong)                       │
│         Rate Limiting │ Auth │ Routing │ SSL                 │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐   ┌─────────────────┐   ┌─────────────────┐
│  AuthService  │   │   UserService   │   │   RideService   │
│    (C#)       │   │      (C#)       │   │      (Go)       │
└───────────────┘   └─────────────────┘   └─────────────────┘
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌─────────────────┐   ┌─────────────────┐
│MatchingService│   │ LocationService │   │ PricingService  │
│     (Go)      │   │      (Go)       │   │      (Go)       │
└───────────────┘   └─────────────────┘   └─────────────────┘
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    MESSAGE BROKER                           │
│                      (RabbitMQ)                             │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                         DATA LAYER                          │
│   PostgreSQL    │    S3     │    MongoDB    │     Redis     │
└─────────────────────────────────────────────────────────────┘
```

Подробная документация по архитектуре: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)

## 🚀 Быстрый старт

### Требования

- Docker 24.0+
- Docker Compose 2.20+
- Node.js 20+ (для frontend)
- Go 1.21+ (для локальной разработки Go сервисов)
- .NET 9 SDK (для локальной разработки C# сервисов)

### Установка и запуск

```bash
# 1. Клонировать репозиторий
git clone https://github.com/Go-Yu-Da/Taxi.git
cd taxi-service

# 2. Скопировать файлы окружения
cp .env.example .env

# 3. Запустить инфраструктуру
docker-compose up -d postgres redis rabbitmq mongodb

# 4. Применить миграции
./scripts/migrate.sh

# 5. Запустить все сервисы
docker-compose up -d

# 6. Проверить статус
docker-compose ps

### Проверка работоспособности

bash
# Health check всех сервисов
curl http://localhost:8000/health

# Auth Service
curl http://localhost:8000/api/auth/health

# Ride Service
curl http://localhost:8000/api/rides/health

### Запуск Frontend

bash
# Passenger App
cd frontend/passenger-app
npm install
npm run dev

# Driver App
cd frontend/driver-app
npm install
npm run dev

# Admin Panel
cd frontend/admin-panel
npm install
npm run dev
```

## 📁 Структура проекта

taxi-service/  
├── services/  
│   ├── api-gateway/          # Go  
│   ├── auth-service/         # C# .NET  
│   ├── user-service/         # C# .NET  
│   ├── ride-service/         # Go  
│   ├── matching-service/     # Go  
│   ├── location-service/     # Go  
│   ├── pricing-service/      # Go  
│   ├── notification-service/ # C# .NET  
│   └── admin-service/        # C# .NET  
├── frontend/  
│   ├── passenger-app/        # SolidJS  
│   ├── driver-app/           # SolidJS  
│   └── admin-panel/          # SolidJS  
├── proto/                    # gRPC протоколы  
├── scripts/                  # Утилиты и скрипты  
├── docs/                     # Документация  
├── docker-compose.yml  
├── .env.example  
└── README.md

## 📚 API документация

После запуска сервисов документация доступна:

| Сервис | Swagger UI |
|--------|------------|
| Auth Service | http://localhost:8085/swagger |
| User Service | http://localhost:8086/swagger |
| Ride Service | http://localhost:8081/swagger |
| Pricing Service | http://localhost:8084/swagger |
| Admin Service | http://localhost:8090/swagger |

Postman коллекция: [docs/postman/TaxiService.postman_collection.json](docs/postman/)
  
  
## 🔧 Разработка

### Локальный запуск отдельного сервиса

```bash
# Go сервис
cd services/ride-service
go run cmd/main.go

# C# сервис
cd services/auth-service
dotnet run
```

### Запуск тестов

```bash
# Go тесты
cd services/ride-service
go test ./...

# C# тесты
cd services/auth-service
dotnet test

# Frontend тесты
cd frontend/passenger-app
npm test
```

### Линтинг

```bash
# Go
golangci-lint run

# C#
dotnet format

# Frontend
npm run lint
```

## 🌐 Переменные окружения

Основные переменные (см. `.env.example`):

```env
# Database
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_PASSWORD=your_password

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# RabbitMQ
RABBITMQ_HOST=localhost
RABBITMQ_PORT=5672
RABBITMQ_USER=admin
RABBITMQ_PASSWORD=admin

# JWT
JWT_SECRET=your_jwt_secret
JWT_EXPIRY=3600

# Google Maps
GOOGLE_MAPS_API_KEY=your_api_key
```

## 👥 Команда

| Роль | Ответственность |
|------|-----------------|
| Team Lead | Архитектура, Go сервисы (Ride, Matching, Location, Pricing), код-ревью, task-managment, SolidJS приложения (Passenger, Driver, Admin), Devops (CI/CD) |
| Backend Developer 1 | Архитектура, C# сервисы (Auth, User, Payment, Notification, Admin) |
| Backend Developer 2 | Архитектура, ... |


## 📄 Лицензия

Apache License 2.0. См. [LICENSE](LICENSE) для деталей.

---

## 🔗 Полезные ссылки

- [Техническая документация](docs/ARCHITECTURE.md)
- [Changelog](CHANGELOG.md)
