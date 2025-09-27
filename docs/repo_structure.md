# Rello Club — Структура репозитория (MVP v1.0)

**Статус:** Черновик (согласовано)

Документ фиксирует структуру кодовой базы для старта разработки: монорепозиторий, пакетирование, зависимости, конфиги, Docker/Compose, CI и правила работы с сабмодулем документации.

---

## 1) Репозитории

- **Код:** отдельный монорепозиторий (например, `rello_platform`).
- **Документация:** `rello_docs` — отдельный репозиторий.
- В код‑репозитории подключаем `rello_docs` как **git submodule** в каталоге `/docs`:
  ```bash
  git submodule add https://github.com/Kinrilans/rello_docs.git docs
  git submodule update --init --recursive
  ```

---

## 2) Монорепо и стандарты

- **Workspaces:** pnpm workspaces (без Turborepo на MVP).
- **Node.js:** v22.12.0 (рекомендуем `.nvmrc`/Volta).
- **Язык:** TypeScript везде, общий `tsconfig.base.json` + project references.
- **Неймспейс пакетов:** `@rello/*`.
- **Код‑стиль:** ESLint + Prettier + Husky (pre-commit с lint-staged).

---

## 3) Дерево каталогов

```
/                 # корень монорепозитория
  apps/
    core/         # @rello/core — финансовое ядро (REST API)
    bots/
      office/     # @rello/bot-office — @RelloOfficeBot
      admin/      # @rello/bot-admin  — @RelloAdminBot
      club/       # @rello/bot-club   — @RelloClubBot
      deal/       # @rello/bot-deal   — @RelloDealBot
      group/      # @rello/bot-group  — @RelloGroupBot
      topics/     # @rello/bot-topics — @RelloTopicsBot
      notice/     # @rello/bot-notice — HTTP-приёмник Notice (без Telegram)
  packages/
    shared/               # @rello/shared — утилиты, HMAC, обёртки HTTP, common types
    clients/
      crypto/             # @rello/client-crypto — провайдер CryptoCurrencyAPI
      aml/                # @rello/client-aml    — провайдер GetBlock (AML)
      fx/                 # @rello/client-fx     — курсы <FIAT>/USDT (mock/реал)
      notice/             # @rello/client-notice — клиент к @RelloNoticeBot HTTP API
    i18n/                 # @rello/i18n — локали RU/EN для ботов
    notice-templates/     # @rello/notice-templates — JSON-шаблоны уведомлений
    errors/               # @rello/errors — источники кодов + автоген типов
  db/
    migrations/           # dbmate SQL-миграции (DDL + эволюция)
    seed/                 # сиды (сервисные кошельки, комиссии, тест-данные)
  infra/
    docker/               # Dockerfile'ы, compose-файлы
    nginx/                # конфиги reverse-proxy (prod)
    deploy/               # скрипты выката/сервисы systemd (по желанию)
  scripts/                # вспомогательные CLI (генерация типов, i18n и т.п.)
  storage/                # локальные артефакты DEV (файлы сделок)
  .github/workflows/      # CI (lint, test, build)
  docs/                   # git submodule → rello_docs (текстовая документация)
```

---

## 4) Границы пакетов и зависимости

- `` зависит от: `@rello/shared`, `@rello/errors`, `@rello/client-crypto`, `@rello/client-aml`, `@rello/client-fx`, `@rello/client-notice`.
- **Боты** (`@rello/bot-*`) зависят от: `@rello/shared`, `@rello/errors`, `@rello/client-fx` (для отображения сумм), при необходимости — `@rello/client-notice`.
- `` — RU/EN словари, экспортирует функции форматирования (учёт правил локализации user/company/group).
- `` — шаблоны Notice (JSON), версии шаблонов.
- `` — из `docs/error_catalog.md` генерирует TS‑типы/enum кодов.

> Зависимости направлены из `apps/*` в `packages/*`. Кросс‑зависимости между `apps/*` **не допускаются** (общие вещи — только через `packages/*`).

---

## 5) Workspaces конфигурация

``** (корень):**

```yaml
packages:
  - "apps/*"
  - "apps/bots/*"
  - "packages/*"
  - "packages/clients/*"
```

``** (корень):**

```json
{
  "name": "@rello/monorepo",
  "private": true,
  "engines": { "node": ">=22.12.0" },
  "scripts": {
    "lint": "eslint .",
    "format": "prettier -w .",
    "db:migrate": "dbmate up",
    "db:down": "dbmate down -1",
    "db:status": "dbmate status",
    "db:seed": "node db/seed/index.js",
    "dev:core": "pnpm --filter @rello/core dev",
    "dev:bots": "pnpm --filter @rello/bot-* dev",
    "dev:notice": "pnpm --filter @rello/bot-notice dev",
    "build": "pnpm -r build",
    "test": "pnpm -r test"
  }
}
```

---

## 6) TypeScript и алиасы

``** (корень):**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "resolveJsonModule": true,
    "outDir": "dist",
    "baseUrl": ".",
    "paths": {
      "@rello/shared/*": ["packages/shared/src/*"],
      "@rello/errors":   ["packages/errors/src/index.ts"],
      "@rello/client-crypto": ["packages/clients/crypto/src/index.ts"],
      "@rello/client-aml":    ["packages/clients/aml/src/index.ts"],
      "@rello/client-fx":     ["packages/clients/fx/src/index.ts"],
      "@rello/client-notice": ["packages/clients/notice/src/index.ts"],
      "@rello/i18n":          ["packages/i18n/src/index.ts"],
      "@rello/notice-templates": ["packages/notice-templates/src/index.ts"]
    }
  }
}
```

Каждое приложение/пакет имеет свой `tsconfig.json`, который **extends** базовый.

---

## 7) Переменные окружения

- Корневой `.env` (см. `dev_env_setup.md`) + `.env` в каждом `apps/*`.
- `.env.example` — в корне и для каждого приложения отдельно.
- Секреты **не коммитим**, храним в Vault/1Password (см. `api_clients_and_security.md`).

---

## 8) Docker/Compose

- **Core:** `infra/docker/Dockerfile.core` (multi‑stage build, Node 22‑alpine).
- **Боты:** базовый образ `infra/docker/Dockerfile.bot.base` + лёгкий Dockerfile‑шаблон с `ARG BOT_PATH`:
  ```Dockerfile
  FROM node:22-alpine AS base
  # … установка deps
  ARG BOT_PATH
  WORKDIR /workspace
  COPY . .
  RUN pnpm -r build
  CMD ["node", "apps/bots/${BOT_PATH}/dist/index.js"]
  ```
- **Compose:**
  - `infra/docker/docker-compose.dev.yml` — Postgres, pgAdmin, локальный запуск.
  - `infra/docker/docker-compose.prod.yml` — Postgres, core, нужные боты, reverse‑proxy.

---

## 9) Reverse‑proxy (nginx)

- Конфиги в `infra/nginx/`.
- Роутинг:
  - `api.rello.example` → `apps/core` (`/v1/*`).
  - `notice.rello.example` → `apps/bots/notice` (HTTP приёмник).
  - Webhooks Telegram (prod) на `https://api.rello.example/bot/<name>/webhook` (или отдельный хост `bots.rello.example`).

---

## 10) CI (GitHub Actions)

`.github/workflows/` (минимум):

- `lint-and-test.yml` — установка pnpm, `pnpm i`, `pnpm lint`, `pnpm test`.
- `build.yml` — сборка `@rello/core` и выбранных ботов, артефакты Docker (optional).
- `db-migrations-check.yml` — dry‑run миграций (dbmate `status` + `up` в ephemeral БД).

---

## 11) Гит‑процесс

- Ветви: `main` (prod), `develop` (интеграция). Все изменения через PR.
- Коммиты: **Conventional Commits** + commitlint; автогенерация CHANGELOG (по желанию).
- `CODEOWNERS`: пока один владелец на весь репозиторий; далее можно детализировать по папкам (боты/ядро/инфра).
- Лицензия: **MIT**. В `README.md` — карта репозитория и ссылки на ключевые документы из `docs/`.

---

## 12) Тесты и фикстуры

- Юнит‑тесты в каждом пакете — `src/**/__tests__/*` (Vitest/Jest).
- E2E для ядра — `apps/core/test/e2e` (Supertest), отдельная БД `rello_test`.
- Сид‑фикстуры для тестов — в `db/seed/`.

---

## 13) Связь с документацией (submodule)

- ERD, модель данных, API, notice/error каталоги — в `docs/` (сабмодуль на `rello_docs`).
- В коде **ссылаемся** на конкретные файлы: `docs/docs/data_model.md`, `docs/docs/api_clients_and_security.md` и т.д.
- Обновление сабмодуля:
  ```bash
  git submodule update --remote --merge
  ```

---

## 14) Скрипты и утилиты

- В `scripts/` размещаем CLI:
  - генерация TS‑типов ошибок из `docs/docs/error_catalog.md` в `packages/errors`;
  - сборка i18n словарей из `packages/i18n`;
  - загрузка notice‑шаблонов в `packages/notice-templates` (валидация схемы JSON).

---

## 15) Контроль соответствия

Перед ревью PR проверяем:

- `pnpm lint`, `pnpm test` — зелёные;
- `dbmate status` — чисто; миграции согласованы с `docs/Data Model.md`;
- зависимости `apps/*` → только через `packages/*`;
- не добавлены секреты/ключи в гит.

---

## 16) Версии и дальнейшие шаги

**MVP v1.0**

- Зафиксирована структура монорепозитория и границы пакетов.
- Добавлены правила Docker/Compose, nginx, CI и работы с сабмодулем документации.

**План на будущее**

- Включить Turborepo для кэширования сборок и пайплайнов.
- Автоген OpenAPI‑клиента для ботов из `financial_core_openapi.yaml` (при актуализации контракта).
- Интегрировать проверку соответствия notice‑шаблонов контракту `notice_contract.md` в CI.

