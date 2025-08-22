# MVP — план работ и итерации (Node.js + Postgres) (v1)

Часовой пояс: **Europe/Warsaw** · Стек: **Node.js** (боты + API), **PostgreSQL**, Redis (опц.), TronWeb (TRC‑20).\
Запуск для закрытого клуба. Активы: **USDT/USDC**, сеть **TRC‑20** на старте (ERC‑20 — в следующей итерации).

---

## 0) Объём MVP (Итерация 1)

**Функции (минимально жизненные):**

- Боты: **@ClubGateBot**, **@DealDeskBot**, **@TreasuryBot** (минимум); **@ArbiterDeskBot** — базовый.
- Режимы сделок: **pass‑through** + **доверительные пары с EOD‑неттингом** (выплата EOD с ручным подтверждением).
- Депозиты/лимиты: Ledger, `k‑фактор` tier’ов (S/M/L/XL), cap per‑deal/open‑exposure, резерв под пары.
- Выплаты: TRC‑20 **ручные/полуавто** (engine формирует payout\_task, отправка после approve).
- Watcher входящих: TRON (min‑confirmations) → триггер pass‑through.
- Allow‑list адресов компаний; шаблоны реквизитов.
- Deeplink: **Short‑Token** (одноразовый, TTL 2–5 мин). JWT — позже.
- Мониторинг/алерты: базовые (health, очередь выплат, stuck‑tx), @OpsAdminBot — уведомления.
- Админ‑настройки: через `.env`/JSON + несколько служебных команд в @OpsAdminBot.

**Что в перспективу (не в MVP):** ERC‑20, авто‑EOD, полноценная веб‑админка, webhooks для интеграторов, расширенный арбитраж.

### 0.1) Принятые решения (дефолт)

- **Репозиторий:** один (single‑repo), без монорепо/турбо.
- **Пакет‑менеджер:** `npm`.
- **Язык:** чистый **JavaScript** (без TypeScript на старте).
- **Боты:** **telegraf**.
- **API:** **Express**.
- **БД/миграции:** **PostgreSQL + Knex**.
- **Блокчейн:** **TronWeb** (TRC‑20).
- **Очереди:** без Redis; задачи выплат в таблице `payout_task` + периодический воркер.
- **Выплаты:** ручной/полуавто approve через @TreasuryBot.
- **Мониторинг:** логи + @OpsAdminBot (Grafana/Prometheus — позже).

---

## 1) Репозиторий и структура (single‑repo)

- Один репозиторий с папками сервисов и миграциями. Без pnpm/монорепо, всё на **npm**.
- Единая точка входа для ботов — \`\`: поднимает все боты (club/deal/treasury/arbiter/ops) и подключает общее middleware (локализация, логирование).

```
/infra        — docker-compose, env-шаблоны
/api          — Express + Knex + PG
  /routes     — deals, treasury, trust
  /db         — knexfile.js, pool.js
  /services   — limits.js, ledger.js
/bots         — telegraf боты (club, deal, treasury, arbiter, ops)
  index.js    — точка входа для всех ботов + общее middleware (i18n/logging)
  /shared     — i18n.json (ключи из словаря)
/watchers     — tron.js (входящие TRC‑20)
/migrations   — *.sql (таблицы)
/scripts      — seed.js (сидинг компаний/пар)
.env.example
package.json
README.md
```

**package.json (скрипты):**

```json
{
  "scripts": {
    "dev:api": "nodemon api/index.js",
    "dev:bots": "nodemon bots/index.js",
    "dev:watchers": "nodemon watchers/tron.js",
    "db:migrate": "knex --knexfile api/db/knexfile.js migrate:latest",
    "db:seed": "node scripts/seed.js",
    "start": "node api/index.js & node bots/index.js & node watchers/tron.js"
  }
}
```

## 2) База данных (минимум)

База данных (минимум)

Таблицы (минимальный состав для MVP):

- `company(id, name, tz, created_at)`
- `member(id, tg_id, name, created_at)`
- `company_member(company_id, member_id, role)` — роли: `ADMIN|TRADER|APPROVER`
- `wallet(id, company_id, network, token, address, verified, created_at)` — allow‑list получателей
- `offer(id, company_id, network, token, amount_token, price, status, created_at)`
- `deal(id, offer_id, maker_company_id, taker_company_id, mode, network, token, amount_token, status, created_at)`
- `trust_pair(id, a_company_id, b_company_id, daily_limit_usd, cutoff_local, tz, reserve_pct, enabled)`
- `trust_session(id, pair_id, session_date, status)` — `open|closed`
- `eod_settlement(id, session_id, payer_company_id, payee_company_id, amount_token, token, network, status, tx_hash)`
- `deposit(id, company_id, token, network, balance_token)`
- `deposit_ledger(id, company_id, type, amount_token, token, ref, created_at)` — типы см. §5
- `payout_task(id, company_id, purpose, network, token, amount_token, to_address, state, tx_hash, created_at)`
- `incoming_tx(id, company_id, network, token, amount_token, from_address, confirmations, status, tx_hash, created_at)`
- `deeplink_ticket(id, company_id, bot, action, token, expires_at, used)`
- `audit_log(id, actor, action, target, before, after, ts)`

Индексы: `(company_id)`, `(status)`, `(created_at)`, `deal(mode,status)`, `payout_task(state)`. Внешние ключи на `company/id`.

## 3) Боты и UX (MVP)

### @ClubGateBot

- Команды: `start`, `profile`, `accounts`, `partners`, `trust_pairs`.
- Флоу: выбрать компанию → заполнить профиль → добавить/верифицировать адреса → просмотреть партнёров → запросить доверительную пару.

### @DealDeskBot

- Команды: `offers`, `offer_new`, `my_deals`.
- Флоу: создать оффер (TRC‑20, USDT/USDC) → принять оффер → выбрать режим: **pass‑through** или **неттинг в EOD** (если есть пара).
- Показы: карточка сделки, дедлайны, статус входящего/выплаты.

### @TreasuryBot

- Команды: `deposit_address`, `payouts`, `eod_today`.
- Флоу: смотреть задачи выплат `payout_task` → **Approve/Send** (для Approver) → показывать `tx_hash`.
- EOD: список net по парам за сегодня → «Отправить выплату» (ручной запуск).

### @ArbiterDeskBot (база)

- Команды: `open_case`, `my_cases`.
- Флоу: открыть спор, приложить материалы; решение шаблонное → проводка в `deposit_ledger`.

## 4) Блокчейн и выплаты (TRC‑20)

- TronWeb + основной RPC (резервный в `.env`).
- **Входящие**: watcher по адресам платформы; условия — правильная сеть/токен, сумма в допуске, `MIN_CONFIRMATIONS`.
- **Pass‑through**: после подтверждений создаётся `payout_task` на контрагента.
- **State machine payout\_task**: `queued → signing → broadcast → confirmed|failed`.
- **EOD**: ручной запуск payout по net‑сумме (см. §3 Treasury).
- Энергия/TRX: держать буфер; при нехватке — ошибка «insufficient energy» и ретрай позже.

## 5) Депозиты и лимиты

- Формула лимита компании: `company_limit_usd = deposit_available_usd × k_base` (по tier S/M/L/XL).
- Cap’ы: `cap_per_deal_usd`, `cap_open_exposure_usd` (по tier). Превышение → запрет сделки/выплаты.
- Резерв под доверительные пары: `reserve_pct` от лимита пары (снимает `deposit_available`).
- `deposit_ledger.type`: `fund`, `hold_open`, `hold_release`, `penalty`, `reserve_trust`, `adjustment`.

## 6) Безопасность и доступ

- Роли: `ADMIN|TRADER|APPROVER`; критичные действия — только **APPROVER**.
- **Allow‑list адресов**: payout — только на верифицированный адрес компании.
- Deeplink **Short‑Token**: одноразовый, TTL 2–5 мин, привязка к `tg_id` и `company_id`.
- Rate‑limit: базовый (на IP/аккаунт) в API/ботах.

## 7) Мониторинг и операционка

- Health: `/healthz` у API и watcher.
- Базовые метрики (логами): начало/окончание payout, очередь `payout_task`, ошибки RPC, время подтверждения.
- @OpsAdminBot: текстовые алерты (SEV2/SEV1) и быстрые кнопки (ACK/SNOOZE/Runbook/Dashboard — можно-заглушки).

## 8) Деплой и инфраструктура (минимум)

- **Локально**: запуск через `nodemon` трёх процессов (api/bots/watchers).
- **VPS**: один сервер (1–2 ГБ RAM), процессы под `pm2` или systemd.
- Postgres локально/на VPS; бэкап—скрипт `pg_dump` раз в день.
- `.env` хранить локально; секреты позже переведём в vault.

## 9) Чек‑лист готовности MVP

- Pass‑through E2E: входящий TRC‑20 → payout confirmed (1 USDT канарейка).
- EOD: 3 сделки в доверительной паре → ручной settlement → payout confirmed.
- Депозиты/лимиты: блокируются превышения; резерв пары учитывается.
- Allow‑list: запрет payout на новый адрес.
- Алерты в @OpsAdminBot приходят; health OK.
- Документация готова (SOP/IRP/Backup).

---

## 10) План работ (2–4 недели, приоритизация)

**Неделя 0 — Dev Kickoff (локалка, без затрат)**

- Создать один репозиторий, `npm init -y`, добавить `.gitignore`.
- Установить пакеты: `express telegraf pg knex dotenv axios tronweb nodemon`.
- Настроить Postgres и `.env`, добавить `knexfile.js`, миграции‑«болванки», выполнить `db:migrate` и `db:seed`.
- Поднять dev‑процессы: `api`, `bots/index.js`, `watchers/tron.js`.

**Неделя 1 — Club/Deal + watcher**

- @ClubGateBot: онбординг, профиль/реквизиты, партнёры, запрос доверительной пары.
- @DealDeskBot: оффер → accept → `deal` (TRC‑20 USDT/USDC), выбор режима.
- Watcher входящих TRC‑20: `min_confirmations` → создание `payout_task`.

**Неделя 2 — Treasury + pass‑through E2E**

- @TreasuryBot: список `payout_task`, approve/send, показ `tx_hash`.
- Депозит‑ledger, лимиты/блокировки (k‑фактор, cap per‑deal/open‑exposure).
- E2E pass‑through: входящий → payout confirmed (канарейка 1 USDT).

**Неделя 3 — Trust/EOD + Arbiter + Ops**

- Trust/EOD: дневная `trust_session`, расчёт net, ручной `eod_settlement`.
- @ArbiterDeskBot: открыть спор, решение `refund/penalty/hold_release` → ledger.
- @OpsAdminBot: базовые алерты (SEV2/SEV1), health‑эндпоинты.

**Неделя 4 — Полировка + пилот**

- Полировка UX/текстов, backup‑скрипт БД, экспорт логов.
- Канареечные выплаты и пилот 2–4 партнёра (≤10% объёма).
- Go/No‑Go по чек‑листу; фиксы и готовность к расширению.

---

## 11) Итерации 2–4 (кратко)

**Итерация 2 (расширение сетей/автоматизация)**

- ERC‑20 (USDT/USDC), gas‑менеджмент, auto‑EOD (engine priority/batching), JWT (RS256), webhooks v1, простая веб‑админка.
- People: 1 **Backend** (Node, 0.5–1 FTE), 0.5 **DevOps** (мониторинг, CI/CD).

**Итерация 3 (операции/качество)**

- Полный арбитраж (шаблоны, апелляции), админ‑консоль (tiers/limits/webhooks/JWKS), отчётность/экспорт.
- Авто‑ребаланс hot/cold (workflow), Staging → Canary CD.
- People: 1 **Frontend** (админка), 1 **QA** (автотесты), 0.5 **Designer** (UX ботов/админки).

**Итерация 4 (масштабирование/анализ)**

- DR‑регион, репликация, отказоустойчивость RPC.
- Аналитика и биллинг/комиссии, партнёрские интеграции.
- People: 1 **Data/Analytics**, 0.5 **Finance Ops**.

---

## 12) Риски и рычаги сокращения объёма

- При дефиците времени/ресурсов:
  - выключить Arbiter до ит.2 (споры через админа и ledger‑проводки);
  - EOD — только вручную, без partial/T+1 автоматики;
  - адреса верифицировать вручную (без микро‑tx);
  - отложить USDC, оставить USDT (TRC‑20) на старте.

---

## 13) Definition of Done (MVP)

- Функции из §0 доступны пилотам и проходят **E2E сценарии** из тест‑плана.
- SLO 7 дней в зелёной зоне на prod‑canary (p95 pass‑through ≤ 15 мин; p95 EOD ≤ 60 мин).
- Ранбуки/алерты/бэкапы соответствуют плану запуска; Go/No‑Go = **GO**.

---

## 14) Роли и нагрузка (MVP)

- **Founder/PM (вы):** приоритизация, деплой, партнёры, запуск пилота.
- **Assistant (я):** ТЗ/спеки, каркасы кода/сниппеты, SQL/миграции, тексты, тест‑кейсы, ревью.
- **Найм на итерации 2–4:** Backend → DevOps → Frontend/QA → Analytics/Finance Ops.

