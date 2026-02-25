# Руководство по Docker-конфигурации

## 1. Обзор

Проект использует `docker-compose.yml` для запуска полного стека:
- Инфраструктура:
  - `postgres` (`postgres:16`)
  - `redis` (`redis:7`)
- Rust-сервисы (собираются из локальных Dockerfile):
  - `api_service`
  - `admin_service`
  - `parser_service`
  - `telegram_service`

Путь к compose-файлу:
- `docker-compose.yml`

Dockerfile сервисов:
- `services/api_service/Dockerfile`
- `services/admin_service/Dockerfile`
- `services/parser_service/Dockerfile`
- `services/telegram_service/Dockerfile`

## 2. Требования

Установите:
- Docker Engine
- Docker Compose plugin (`docker compose`)

Проверка:

```bash
docker --version
docker compose version
```

## 3. Карта сервисов

`postgres`:
- Контейнер: `vsu_postgres`
- Проброс порта: `5432:5432`
- Том: `postgres_data:/var/lib/postgresql/data`
- Настройки БД:
  - `POSTGRES_USER=admin`
  - `POSTGRES_PASSWORD=admin`
  - `POSTGRES_DB=vsu_db`
- Healthcheck: `pg_isready -U admin -d vsu_db`

`redis`:
- Контейнер: `vsu_redis`
- Проброс порта: `6379:6379`
- Healthcheck: `redis-cli ping`

`api_service`:
- Контейнер: `vsu_api_service`
- Контекст сборки: корень проекта (`.`)
- Dockerfile: `services/api_service/Dockerfile`
- Проброс порта: `8080:8080`
- Переменные окружения:
  - `DATABASE_URL=postgres://admin:admin@postgres:5432/vsu_db`
  - `REDIS_URL=redis://redis:6379`
  - `RUST_LOG=info`
- Зависит от здоровых `postgres` и `redis`

`admin_service`:
- Контейнер: `vsu_admin_service`
- Контекст сборки: корень проекта (`.`)
- Dockerfile: `services/admin_service/Dockerfile`
- Проброс порта: `8081:8081`
- Переменные окружения:
  - `DATABASE_URL=postgres://admin:admin@postgres:5432/vsu_db`
  - `REDIS_URL=redis://redis:6379`
  - `RUST_LOG=info`
- Зависит от здоровых `postgres` и `redis`

`parser_service`:
- Контейнер: `vsu_parser_service`
- Контекст сборки: корень проекта (`.`)
- Dockerfile: `services/parser_service/Dockerfile`
- Переменные окружения:
  - `DATABASE_URL=postgres://admin:admin@postgres:5432/vsu_db`
  - `REDIS_URL=redis://redis:6379`
  - `RUST_LOG=info`
- Зависит от здоровых `postgres` и `redis`

`telegram_service`:
- Контейнер: `vsu_telegram_service`
- Контекст сборки: корень проекта (`.`)
- Dockerfile: `services/telegram_service/Dockerfile`
- Переменные окружения:
  - `DATABASE_URL=postgres://admin:admin@postgres:5432/vsu_db`
  - `REDIS_URL=redis://redis:6379`
  - `RUST_LOG=info`
  - `TELEGRAM_TOKEN` (передается через `.env`)
- Зависит от здоровых `postgres` и `redis`

## 4. Шаблон Dockerfile для Rust-сервисов

Все Dockerfile сервисов используют multi-stage сборку:

1. Этап `builder` (`rust:1.75`)
- Копирует весь workspace (`COPY . .`)
- Собирает нужный пакет: `cargo build --release -p <service_name>`

2. Runtime-этап (`debian:bookworm-slim`)
- Устанавливает `ca-certificates`
- Копирует бинарник из builder-этапа
- Запускает бинарник как команду контейнера

Почему нужен `COPY` всего workspace:
- Сервисы зависят от crates в `crates/*`
- Cargo должен видеть весь workspace для разрешения локальных зависимостей

## 5. Переменные окружения

`docker-compose.yml` задает большую часть переменных напрямую.

Для секретов (например, токен Telegram) используйте `.env` в корне проекта.

Рекомендуемый пример `.env`:

```dotenv
TELEGRAM_TOKEN=your_real_token_here
```

Важно по безопасности:
- Не коммитьте реальные токены/пароли.
- Если токен был раскрыт, перевыпустите его в исходной системе и обновите `.env`.

## 6. Запуск и остановка

Запуск всех сервисов в фоне:

```bash
docker compose up -d --build
```

Запуск в foreground с логами:

```bash
docker compose up --build
```

Остановить контейнеры:

```bash
docker compose stop
```

Остановить и удалить контейнеры/сеть:

```bash
docker compose down
```

Остановить и удалить контейнеры/сеть и именованные тома:

```bash
docker compose down -v
```

## 7. Сборка

Собрать все образы:

```bash
docker compose build
```

Собрать один сервис:

```bash
docker compose build api_service
docker compose build admin_service
docker compose build parser_service
docker compose build telegram_service
```

Пересборка без кеша:

```bash
docker compose build --no-cache
```

## 8. Логи и диагностика

Логи всех сервисов:

```bash
docker compose logs -f
```

Логи конкретного сервиса:

```bash
docker compose logs -f api_service
docker compose logs -f admin_service
docker compose logs -f parser_service
docker compose logs -f telegram_service
docker compose logs -f postgres
docker compose logs -f redis
```

Статус сервисов:

```bash
docker compose ps
```

Проверка healthcheck:

```bash
docker inspect --format='{{json .State.Health}}' vsu_postgres
docker inspect --format='{{json .State.Health}}' vsu_redis
```

## 9. Доступ к сервисам и портам

Доступ с хоста:
- API-сервис: `http://localhost:8080`
- Admin-сервис: `http://localhost:8081`
- Postgres: `localhost:5432`
- Redis: `localhost:6379`

Внутренние хосты Docker-сети (используются сервисами):
- Postgres: `postgres`
- Redis: `redis`

## 10. Частые операции

Перезапуск одного сервиса:

```bash
docker compose restart api_service
```

Пересоздание одного сервиса после правок Dockerfile/env:

```bash
docker compose up -d --build --no-deps api_service
```

Вход в контейнер:

```bash
docker compose exec api_service sh
docker compose exec postgres psql -U admin -d vsu_db
docker compose exec redis redis-cli
```

## 11. Troubleshooting

Порт уже занят:
- Измените host-порт в `docker-compose.yml` или освободите локальный процесс.

Сервис сразу завершается:
- Проверьте логи: `docker compose logs -f <service>`
- Убедитесь, что имя бинарника в Dockerfile совпадает с именем cargo-пакета.

`telegram_service` не стартует:
- Проверьте, что `TELEGRAM_TOKEN` задан и валиден.

Ошибки подключения к DB/Redis:
- Убедитесь, что `postgres` и `redis` в статусе healthy (`docker compose ps`).
- Используйте внутренние хосты `postgres` и `redis`, а не `localhost`.

Долгая сборка:
- Нормально для первой сборки (компилируются зависимости Rust workspace).
- Повторные `docker compose build` используют слой кеша.

## 12. Чеклист при добавлении нового сервиса

1. Добавить пакет сервиса в `services/<new_service>/`.
2. Создать `services/<new_service>/Dockerfile` по текущему шаблону.
3. Добавить сервис в `docker-compose.yml`:
   - `build.context: .`
   - `build.dockerfile: services/<new_service>/Dockerfile`
   - нужные `environment`
   - `depends_on` с healthcheck-зависимостями при необходимости
4. Пересобрать и запустить:
   - `docker compose up -d --build <new_service>`
