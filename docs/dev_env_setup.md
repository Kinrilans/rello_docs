# Rello Club — Настройка DEV‑окружения (MVP v1.0)

**Статус:** Черновик (согласовано)

Документ описывает, как быстро развернуть локальное окружение для разработки финансового ядра и ботов Rello Club.

---

## 1) Быстрый старт (5 минут)
1. **Предусловия**: Node.js **v22.12.0**, pnpm, Docker + Docker Compose, Git.
2. Клонируйте репозиторий:
   ```bash
   git clone https://github.com/Kinrilans/rello_docs.git
   cd rello_docs
   ```
3. Установите зависимости монорепозитория:
   ```bash
   pnpm i
   ```
4. Поднимите инфраструктуру БД:
   ```bash
   docker compose -f docker-compose.dev.yml up -d postgres pgadmin
   ```
5. Подготовьте БД: примените миграции и сид‑данные:
   ```bash
   pnpm db:migrate
   pnpm db:seed
   ```
6. Запустите ядро и выбранные боты (long‑polling):
   ```bash
   pnpm dev:core
   ENABLE_DEAL=true ENABLE_TOPICS=true pnpm dev:bots
   ```
7. Прогоните smoke‑тест:
   ```bash
   curl -sS http://localhost:8080/health | jq
   ```

> Все команды и порты настраиваются через `.env` (см. §5).

---

## 2) Требования
- **Node.js:** v22.12.0 (рекомендуем nvm/Volta для фикса версии)
- **Пакетный менеджер:** pnpm
- **PostgreSQL:** v18 (через Docker)
- **ОС:** Linux/macOS/Windows (WSL2)

---

## 3) Структура монорепозитория
```
/ (корень)
  db/
    migrations/         # SQL‑миграции (dbmate)
    seed/               # сид‑скрипты/данные
  packages/
    shared/             # общие утилиты (i18n, ошибки, HMAC, клиенты)
  core/                 # финансовое ядро (REST API)
  bots/
    office/             # @RelloOfficeBot
    admin/              # @RelloAdminBot
    club/               # @RelloClubBot
    deal/               # @RelloDealBot
    group/              # @RelloGroupBot
    topics/             # @RelloTopicsBot
    notice/             # @RelloNoticeBot (HTTP‑приёмник Notice)
  storage/              # локальное хранилище артефактов DEV
  docker-compose.dev.yml
```

---

## 4) Инструменты и режимы
- **Боты:** `grammY` + `@grammyjs/keyv` (файловый адаптер) — для сессий в DEV.
- **Запуск:** DEV — **long‑polling**; PROD — webhooks.
- **БД/миграции:** `dbmate` (сырой SQL), каталог `db/migrations`.
- **Курсы валют (FX):** в DEV по умолчанию **mock**, можно переключить на реальный провайдер.
- **Ончейн/AML:** в DEV по умолчанию **mock** обоих провайдеров.

---

## 5) Переменные окружения
### 5.1 Корневой `.env` (пример)
```dotenv
# Режим и время
NODE_ENV=development
TZ=UTC

# Порты сервисов
CORE_HTTP_PORT=8080
NOTICE_HTTP_PORT=8081

# База данных
DB_HOST=localhost
DB_PORT=5432
DB_USER=rello
DB_PASSWORD=rello
DB_NAME=rello_dev
DATABASE_URL=postgres://rello:rello@localhost:5432/rello_dev?sslmode=disable

# dbmate
DBMATE_DATABASE_URL=${DATABASE_URL}
DBMATE_MIGRATIONS_DIR=./db/migrations
DBMATE_SCHEMA_FILE=./db/schema.sql

# Провайдеры (DEV по умолчанию мок)
MOCK_PROVIDERS=true

# Crypto provider (реальный включать при необходимости)
CRYPTO_PROVIDER=new.cryptocurrencyapi.net
CRYPTO_API_KEY=

# AML provider (реальный включать при необходимости)
AML_PROVIDER=getblock
AML_API_KEY=

# FX‑курсы: пары <любой фиат>/USDT
FX_PROVIDER=mock   # mock | binance | <другой поддержанный>
FX_CACHE_TTL_SEC=30
# Стратегия разрешения: прямой курс или кросс через USD
FX_RESOLVE_STRATEGY=direct_or_cross

# Notice (ядро -> NoticeBot HTTP API)
NOTICE_API_URL=http://localhost:${NOTICE_HTTP_PORT}
NOTICE_API_KEY=dev-notice-key
NOTICE_API_SECRET=dev-notice-secret

# Файлы DEV
STORAGE_DIR=storage

# Включение ботов
ENABLE_OFFICE=false
ENABLE_ADMIN=false
ENABLE_CLUB=false
ENABLE_DEAL=true
ENABLE_GROUP=false
ENABLE_TOPICS=true
ENABLE_NOTICE=true
```

### 5.2 `.env` каждого бота (пример)
Общие ключи + Telegram‑токен и язык по умолчанию для сообщений об ошибках.

```dotenv
# @RelloDealBot (.env в bots/deal)
TELEGRAM_BOT_TOKEN=
BOT_LONG_POLLING=true
WEBHOOK_BASE_URL=
TELEGRAM_WEBHOOK_SECRET=

# HMAC для вызовов ядра
API_KEY=deal-dev-key
API_SECRET=deal-dev-secret

# Локаль ошибок по умолчанию (может переопределяться Accept-Language)
ERROR_LOCALE_DEFAULT=en
```

> Для остальных ботов переменные аналогичны: свой `TELEGRAM_BOT_TOKEN`, свои `API_KEY/SECRET`. @RelloNoticeBot поднимает **HTTP‑приёмник** (без Telegram), принимает конверты из ядра.

---

## 6) Docker Compose (DEV)
`docker-compose.dev.yml` (фрагмент):
```yaml
services:
  postgres:
    image: postgres:18
    environment:
      POSTGRES_DB: rello_dev
      POSTGRES_USER: rello
      POSTGRES_PASSWORD: rello
      TZ: UTC
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U rello -d rello_dev"]
      interval: 5s
      timeout: 5s
      retries: 10

  pgadmin:
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: dev@rello.local
      PGADMIN_DEFAULT_PASSWORD: devpass
    ports: ["8089:80"]
    depends_on: [postgres]

volumes:
  pgdata:
```

> При отсутствии образа `postgres:18` используйте ближайший доступный LTS (например, `postgres:16`), но для PROD мы целимся в 18.

---

## 7) Миграции и сиды
- Применение миграций:
  ```bash
  pnpm db:migrate   # dbmate up
  ```
- Откат последней миграции:
  ```bash
  pnpm db:down
  ```
- Статус миграций:
  ```bash
  pnpm db:status
  ```
- Сиды (минимум): сервисный владелец `service`, кошельки `main/dirty`, стартовые комиссии, тестовая компания и рабочая группа.
  ```bash
  pnpm db:seed
  ```

---

## 8) Запуск сервисов
### 8.1 Ядро
```bash
pnpm dev:core
# сервер слушает http://localhost:${CORE_HTTP_PORT}
```

### 8.2 Боты (long‑polling)
```bash
# включайте нужные флаги в .env или командой
ENABLE_DEAL=true ENABLE_TOPICS=true pnpm dev:bots
```

### 8.3 NoticeBot (HTTP)
```bash
pnpm dev:notice
# слушает http://localhost:${NOTICE_HTTP_PORT}
```

---

## 9) Настройка Telegram (DEV)
- Создайте ботов в BotFather, заполните `TELEGRAM_BOT_TOKEN`.
- Для чат‑групп добавьте соответствующие боты и дайте права администратора по необходимости.
- В DEV используем **long‑polling**; webhooks настроим для PROD.

---

## 10) Мок‑режим провайдеров и FX‑курсы
- `MOCK_PROVIDERS=true` — ядро и боты используют фиктивные ответы для
  - AML‑проверок,
  - ончейн‑инспекции/отправок,
  - FX‑курсов.
- Переключение на реальный курс: `FX_PROVIDER=binance` (или иной поддержанный), стратегия `direct_or_cross`:
  - прямой курс `<FIAT>/USDT`,
  - если прямого нет — кросс через `USD`.

---

## 11) Линтинг, форматирование, тесты
```bash
pnpm lint     # ESLint
pnpm format   # Prettier
pnpm test     # unit (Vitest/Jest), e2e (Supertest) — минимум smoke
```

---

## 12) Тест «окончание установки»
```bash
# 1) Здоровье ядра
curl -sS http://localhost:8080/health | jq

# 2) Пробный запрос с HMAC (см. api_clients_and_security.md)
# 3) Инициируйте тестовую сделку и проверьте уведомления через NoticeBot (mock)
```

---

## 13) Устранение проблем (FAQ)
- **`ECONNREFUSED postgres`** — проверьте, что контейнер `postgres` поднят и на 5432, дождитесь healthcheck.
- **`dbmate: no database URL`** — проверьте `DATABASE_URL`/`DBMATE_DATABASE_URL`.
- **`telegram 401`** — неверный `TELEGRAM_BOT_TOKEN`.
- **`signature invalid`** — сверяйте канонизацию и секрет (см. `api_clients_and_security.md`).
- **Нет курса для валюты** — в DEV включите `FX_PROVIDER=mock`; в реальном провайдере используйте стратегию `direct_or_cross`.

---

## 14) Приложение: рекомендованные `package.json` скрипты (root)
```json
{
  "scripts": {
    "lint": "eslint .",
    "format": "prettier -w .",
    "db:migrate": "dbmate up",
    "db:down": "dbmate down -1",
    "db:status": "dbmate status",
    "db:seed": "node db/seed/index.js",
    "dev:core": "pnpm --filter @rello/core dev",
    "dev:bots": "pnpm --filter @rello/bots... dev",
    "dev:notice": "pnpm --filter @rello/bots-notice dev"
  }
}
```

> Нотация `--filter` требует корректных `package.json` в подпроектах; именование пакетов — на ваше усмотрение.

---

## 15) Версии и изменения
**MVP v1.0**
- Node v22.12.0, pnpm, PostgreSQL v18 (Docker).
- dbmate (SQL‑миграции), mock‑режим провайдеров по умолчанию.
- Long‑polling в DEV, webhooks — для PROD.
- FX: разрешение пар `<FIAT>/USDT` (direct → cross через USD), кэш TTL.

