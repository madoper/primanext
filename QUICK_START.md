# 🚀 Quick Start Guide — Запуск Прима-NEXT за 15 минут

## 📋 Требования

**Минимально:**
- Docker & Docker Compose 3.9+
- 8 GB RAM
- 50 GB свободного места

**Для разработки (опционально):**
- .NET 8.0 SDK
- Node.js 20+
- Visual Studio Code / Visual Studio 2022

---

## ⚡ Вариант 1: Docker Compose (все в одном)

### Шаг 1: Клонируем репозиторий

```bash
git clone https://github.com/primanext/primanext.git
cd primanext
```

### Шаг 2: Создаем .env файл

```bash
cp .env.example .env
# Отредактируйте пароли в .env если нужно
```

### Шаг 3: Запускаем контейнеры

```bash
docker-compose up -d

# Проверяем статус
docker-compose ps
```

### Шаг 4: Ждем инициализации БД (~2-3 минуты)

```bash
# Смотрим логи инициализации
docker-compose logs -f postgres
docker-compose logs -f mongodb
```

### Шаг 5: Открываем в браузере

| Сервис | URL | Логин |
|--------|-----|-------|
| 🌐 **Frontend** | http://localhost:3000 | — |
| 🔌 **API Gateway** | http://localhost:8000/swagger | — |
| 📊 **Grafana** | http://localhost:3001 | admin / admin |
| 📈 **Prometheus** | http://localhost:9090 | — |
| 🐰 **RabbitMQ** | http://localhost:15672 | guest / guest |
| 🍃 **MongoDB** | mongodb://localhost:27017 | — |
| 🐘 **PostgreSQL** | postgresql://localhost:5432 | — |
| 🔴 **Redis** | redis://localhost:6379 | — |

### Шаг 6: Тестируем API

```bash
# Регистрация
curl -X POST http://localhost:8000/api/users/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "Password123!",
    "firstName": "Test",
    "lastName": "User"
  }'

# Авторизация
curl -X POST http://localhost:8000/api/users/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "Password123!"
  }'

# Поиск компании по ИНН
curl -X GET "http://localhost:8000/api/search?q=7701102700" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

---

## 📦 Вариант 2: Kubernetes (Production-like)

### Требования

- kubectl 1.27+
- Kubernetes кластер (minikube, Docker Desktop K8s, или облако)
- 16+ GB RAM

### Установка

```bash
# 1. Создаем namespace
kubectl create namespace primanext

# 2. Создаем secrets
kubectl create secret generic primanext-secrets \
  --from-literal=postgres-password=your_secure_password \
  --from-literal=mongo-password=your_secure_password \
  -n primanext

# 3. Деплоим конфиги
kubectl apply -f k8s/configmaps/ -n primanext
kubectl apply -f k8s/secrets/ -n primanext

# 4. Деплоим базы данных
kubectl apply -f k8s/databases/ -n primanext

# 5. Ждем готовности БД (2-5 минут)
kubectl wait --for=condition=ready pod -l app=postgres -n primanext --timeout=300s

# 6. Деплоим приложение
kubectl apply -f k8s/services/ -n primanext

# 7. Проверяем статус
kubectl get pods -n primanext -w
kubectl get svc -n primanext

# 8. Открываем port-forward
kubectl port-forward -n primanext svc/api-gateway-service 8000:80
kubectl port-forward -n primanext svc/frontend-service 3000:80

# 9. Открываем в браузере
# http://localhost:3000 (frontend)
# http://localhost:8000/swagger (API)
```

---

## 🔧 Вариант 3: Локальная разработка (без Docker)

### Backend (.NET)

```bash
cd backend

# Восстанавливаем зависимости
dotnet restore

# Создаем БД (нужна PostgreSQL локально)
dotnet ef database update -p src/Services/PrimaNext.Company.API

# Запускаем dev-сервер
dotnet watch run --project src/ApiGateway/

# API будет доступен на http://localhost:5000
```

### Frontend (Vue 3)

```bash
cd frontend

# Устанавливаем зависимости
npm install

# Запускаем dev-сервер
npm run dev

# Frontend будет доступен на http://localhost:5173
```

---

## ✅ Проверка работы

### 1. Проверяем здоровье сервисов

```bash
# Docker Compose
curl http://localhost:8000/health/ready
curl http://localhost:8000/health/live

# Kubernetes
kubectl get pods -n primanext
kubectl logs -n primanext -l app=api-gateway
```

### 2. Проверяем БД

```bash
# PostgreSQL
docker exec primanext-postgres psql -U postgres -d primanext -c "SELECT COUNT(*) FROM pg_tables;"

# MongoDB
docker exec primanext-mongodb mongosh --eval "db.adminCommand('ping')"

# Redis
docker exec primanext-redis redis-cli ping

# Elasticsearch
curl http://localhost:9200/_cluster/health
```

### 3. Проверяем API

```bash
# Получить список сервисов
curl http://localhost:8000/swagger

# Проверить Swagger UI
open http://localhost:8000/swagger/ui
```

---

## 🐛 Troubleshooting

### Порты уже заняты

```bash
# Найти процесс на порту
lsof -i :8000
lsof -i :3000

# Завершить процесс
kill -9 <PID>

# Или изменить порты в .env
FRONTEND_PORT=3001
API_GATEWAY_PORT=8001
```

### Недостаточно памяти

```bash
# Уменьшить лимиты в docker-compose.yml
# Найти section "deploy.resources.limits"

# Или удалить ненужные сервисы
docker-compose up -d --scale neo4j=0
```

### Ошибки подключения к БД

```bash
# Проверить логи
docker-compose logs postgres
docker-compose logs mongodb

# Пересоздать контейнеры
docker-compose down -v
docker-compose up -d

# Проверить переменные в .env
cat .env | grep -E "POSTGRES|MONGO|REDIS"
```

### Frontend не загружается

```bash
# Проверить консоль браузера (F12)
# Проверить переменные окружения
cat frontend/.env

# Пересоздать build
docker-compose up -d --build frontend
```

---

## 📊 Что дальше?

### 1️⃣ Создайте тестовую компанию

```bash
# Используйте форму на http://localhost:3000
# Или через API:
curl -X POST http://localhost:8000/api/companies \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "inn": "7701102700",
    "ogrn": "1177746602985"
  }'
```

### 2️⃣ Посмотрите примеры

- Frontend: `/frontend/src/views/examples/`
- Backend: `/backend/src/Services/PrimaNext.Company.API/Examples/`
- API: http://localhost:8000/swagger

### 3️⃣ Читайте полную документацию

- [Backend Spec](./docs/backend/01-backend-overview.md)
- [Frontend Spec](./docs/frontend/01-frontend-overview.md)
- [Deployment Guide](./docs/devops/01-deployment-guide.md)

### 4️⃣ Запустите тесты

```bash
# Backend
dotnet test backend/

# Frontend
npm run test frontend/

# E2E
npm run test:e2e frontend/
```

---

## 🆘 Нужна помощь?

- 📖 Документация: https://docs.primanext.ru
- 💬 Slack: #development
- 🐛 Issues: https://github.com/primanext/primanext/issues
- 📧 Email: dev-team@primanext.ru

---

## ⏱️ Примерное время запуска

| Компонент | Время |
|-----------|--------|
| Docker Compose up | 10-20 сек |
| PostgreSQL init | 30-60 сек |
| MongoDB init | 20-30 сек |
| Backend ready | 30-60 сек |
| Frontend ready | 20-30 сек |
| **Итого** | **~3-5 минут** |

**Важно**: Первый запуск медленнее из-за скачивания образов и инициализации БД.

---

## 🎉 Готово!

Ваша локальная копия Прима-NEXT готова к разработке и тестированию!

**Следующие шаги:**
1. Создайте новую ветку: `git checkout -b feature/your-feature`
2. Внесите изменения в код
3. Запушьте ветку: `git push origin feature/your-feature`
4. Создайте Pull Request

Happy coding! 🚀
