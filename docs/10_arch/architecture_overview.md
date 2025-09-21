# Rello Club — Architecture Overview (MVP v1.0)

**Status:** Draft (approved baseline decisions)

## 0) Purpose & Scope
Короткий обзор архитектуры MVP Rello Club: контекст (C4 L1), контейнеры (L2), ключевые компоненты (L3), внешние зависимости, правила интеграции, безопасность, эксплуатация. Детализация API, схем данных и state‑машин — в отдельных документах.

**Языки UI:** RU/EN · **TZ:** Europe/Warsaw · **Сеть/токен:** TRC20 · **Внутр. валюта:** RC = 1:1 USD/USDT/USDC · **Фиат интеграций:** нет (MVP RC↔RC)

---

## 1) Context (C4 — Level 1)
**Акторы:**
- **Клиент** (покупатель / продавец), **Работник** (делегат), **Админ/Модератор**, **Владелец сервиса**.
- **Telegram** (чаты/группы/боты), **CryptoCurrencyAPI** (кошельки/транзакции TRC20), **GetBlock** (AML / риск адресов), **Рейты** (EUR/USDT, напр. Binance public API).

**Высокоуровневые цели:**
- Пополнение/вывод RC, сделки B2C/C2B, модерация и арбитраж, уведомления, рефералка.

---

## 2) Containers (C4 — Level 2)
### 2.1 Services (микросервисы/боты)
- **financial-core** — бизнес‑ядро денег и сделок: леджер RC, заморозки/переводы, шлюз к CryptoCurrencyAPI, AML через GetBlock, модуль курсов, комиссии, вычисления, правила и NFR. HTTP API (JSON), HMAC‑подпись, идемпотентность, корреляция.
- **club-bot (@RelloClubBot)** — кабинет клиента: регистрация, кошелёк RC/депозит, заявки, история (CSV), рабочие группы/работники, рефералка.
- **deal-bot (@RelloDealBot)** — мастер одиночных сделок (из объявлений Topics): B2C/C2B, сбор реквизитов, заморозка RC, файлы, арбитраж.
- **group-bot (@RelloGroupBot)** — работа в чат‑группах: привязка к продавцу, групповой кошелёк RC, команды `/deal_*`, `/send_*`, `/cash_*`, `/balance`, `/stats_*` и т.п.
- **admin-bot (@RelloAdminBot)** — модерация объявлений, арбитраж, кошелёк RC, депозит, выгрузки.
- **office-bot (@RelloOfficeBot)** — главный админ: комиссии сервиса, пользователи/роли, депозиты/тиры, рефералка, ручные проводки, выгрузки.
- **notice-bot (@RelloNoticeBot)** — доставка *только новых* сообщений пользователям (идемпотентность по `notice_id`, ретраи).
- **topics-bot (@RelloTopicsBot)** — объявления/темы, источник для одиночных сделок.

### 2.2 Shared components
- **Postgres** (один инстанс на VDS, **две логические области**):
  - `financial_core` — деньги/леджер/сделки (изолированные права, доступ только у ядра);
  - `bots` — кэш/сессии/служебные таблицы ботов.
- **Redis** — rate‑limits, токены/TTL, кэш курсов, идемпотентность, очередь ретраев NoticeBot.
- **Nginx (reverse proxy)** — TLS, вебхуки Telegram, IP‑allowlist, rate‑limit.

### 2.3 External
- **CryptoCurrencyAPI** — TRC20 кошельки/ввод/вывод.
- **GetBlock** — AML/риск адресов и возвраты high‑risk.
- **Рейты** — публичный источник EUR/USDT (напр., Binance public API).

---

## 3) Components (C4 — Level 3, ключевые блоки)
### 3.1 financial-core
- **Ledger RC** — счета/балансы/движения RC (double‑entry модель; эскроу/hold).
- **Deals** — state‑машины B2C/C2B (подготовка → подтверждения → успех/арбитраж/отмена).
- **Wallets Gateway** — интеграция с CryptoCurrencyAPI: создание адресов, приём TRC20, вывод, статусы.
- **AML Gateway** — GetBlock: проверка адресов; high‑risk → отказ/возврат (минус комиссия), «грязный» кошелёк.
- **Rates** — EUR/USDT, кэш 30–60 с, фолбэк; конвертация EUR↔RC (RC ≡ USDT).
- **Fees Engine** — приоритет комиссий, расчёт нетто/брутто: `сумма − комиссия клиента − комиссия сервиса`.
- **Idempotency** — Redis, TTL 24ч; ключ = `Idempotency-Key`.
- **Auth** — HMAC (API‑Key + signature + timestamp/nonce); ротация ключей 90 дней.
- **Correlation** — `X-Correlation-Id` во всех логах/ответах.
- **API Server** — HTTP/JSON; ошибки с кодами и `correlation_id`.

### 3.2 Bots (общие компоненты)
- **Command Router + Screens** — правило UI: верхний уровень → новое сообщение; формы → редактирование; успех → инвалидировать/удалять старые экраны.
- **Sessions/Cache** — Postgres (схема `bots`) + Redis.
- **Core Client** — вызовы financial‑core по HMAC; `Idempotency-Key` на записи; `X-Correlation-Id` всегда.
- **Notice Client** — формирование событий, `notice_id`.

### 3.3 notice-bot
- **Inbox API** (от сервисов)
- **De‑dupe Store** (по `notice_id`)
- **Retry Queue** (Redis; 5/30/180 с; 3 попытки)
- **Delivery** (Telegram send; локализация)

---

## 4) Interfaces (высокоуровнево)
**Боты → Core (HTTP/JSON):**
- Auth: `X-Api-Key`, `X-Signature`, `X-Nonce`, `X-Timestamp` (UTC), `X-Correlation-Id`.
- Идемпотентность: `Idempotency-Key` на POST/PUT/PATCH.
- Категории: кошельки/балансы/холды; пополнение/вывод; сделки; комиссии; AML; отчёты; служебные.

**Core → внешние провайдеры:**
- CryptoCurrencyAPI: кошельки/tx; GetBlock: AML; Рейты: публичный HTTP.

**Сервисы → NoticeBot:**
- `notice_id`, тип события, локаль/плейсхолдеры, получатели (user/group/role), `correlation_id`.

---

## 5) Data & Storage
- **Postgres**: один инстанс, 2 схемы (`financial_core`, `bots`); права доступа изолируются на уровне ролей БД.
- **Redis**: TTL‑данные (токены, idempotency, курсы), очереди.
- **Backups (минимум)**: ежедневный dump Postgres (stage/prod раздельно).

---

## 6) Security
- **Transport**: HTTPS (Let’s Encrypt), Telegram Webhook за Nginx; IP‑allowlist Telegram.
- **Service Auth**: HMAC + ротация ключей 90 дней; allowlist IP на Nginx.
- **Secrets**: ENV‑переменные, закрытый `.env` только на сервере; диск VDS с шифрованием.
- **Logs**: структурные JSON, маскирование токенов/адресов/кошельков по regex.

---

## 7) Observability
- **/health** у каждого сервиса.
- **Логи**: уровни info/warn/error; в каждом сообщении — `correlation_id`.
- **Метрики (минимум)**: количество ошибок ядра, среднее время ответов ядра, длина очереди NoticeBot, доля AML‑отказов.
- **Алерты (минимум)**: недоступность /health, ошибки ядра > порога, очередь уведомлений > порога.

---

## 8) Environments & Deploy
- **Окружения**: `stage`, `prod` (разные VDS, токены, БД).
- **Деплой**: Docker Compose: `financial-core`, боты, `postgres`, `redis`, `nginx`.
- **Webhooks**: `https://api.<domain>/webhook/{club|deal|group|admin|office|notice|topics}`.

---

## 9) Business Rules (ключевые)
- **Комиссии (приоритет):** 1) глобальные сервиса (OfficeBot), 2) клиентская для групп (дефолт/настройка в ClubBot), 3) клиентская в объявлениях (Topics/Deal). Итог: `сумма − комиссия клиента − комиссия сервиса`.
- **RC = 1 USD/USDT/USDC** — внутренняя расчётная единица.
- **Курсы:** EUR/USDT из публичного API, кэш 30–60 с; фолбэк на последнее значение.
- **After‑restore lock:** 24 ч блок на сделки и ввод/вывод.
- **Токены/лимиты:** TTL на групповые токены; rate‑limits генерации/команд.

---

## 10) Key Flows (sequence — кратко)
1) **RC Top‑Up (TRC20):** выдача адреса → входящий tx → AML (GetBlock) → OK: зачисление RC; High‑risk: отказ/возврат (минус комиссия) → перевод «грязных» остатков.
2) **RC Withdraw:** заявка → проверка лимитов → исходящий tx (CryptoCurrencyAPI) → статус OK/Fail.
3) **B2C Deal:** объявление → инициация → холд RC у плательщика → подтверждения по ветке → успех (перевод RC) / арбитраж / отмена (разморозка).
4) **C2B Deal:** аналогично, но контрольная сторона обратная.
5) **Group Linking:** токен → привязка к продавцу → создание группового RC‑кошелька → команды `/deal_*` и пр.
6) **Notice Delivery:** сервис генерит событие с `notice_id` → NoticeBot дедуплицирует → ретраи при сбое → доставка.

---

## 11) NFR (минимальные для MVP)
- **Доступность ядра:** 99% мес (MVP). Таймауты HTTP: 5–10 с; ретраи на клиенте ботов (экспоненциально, до 2–3 раз).
- **Производительность:** 50–200 rps по ядру достаточно для старта; задержка вычислений < 300 мс (без внешних вызовов).
- **Стабильность курсов:** устаревание не более 60 с; пометка «курс устарел», если фолбэк.
- **Безопасность:** ротация ключей, маскирование логов, ограничение прав БД.

---

## 12) Decisions (ADR summary)
- SVC interaction: HTTP/JSON; без брокера событий (MVP).
- Storage: один Postgres‑инстанс, 2 схемы; Redis обязателен.
- Auth: API‑Key + HMAC; ротация 90 дней; IP‑allowlist на Nginx.
- Idempotency: `Idempotency-Key` (TTL 24 ч, Redis).
- Trace: `X-Correlation-Id` везде.
- Deploy: Docker Compose на 2 VDS (stage/prod), Nginx TLS.
- Rates: публичный API (Binance) + кэш 30–60 с.
- Комиссии: источник правды — OfficeBot; порядок приоритета зафиксирован.

---

## 13) Out of Scope (MVP)
- Банковские интеграции/фиат платёжки.
- mTLS/JWT, брокер событий, Kubernetes (эволюция возможна позже).
- Расширенный мониторинг/алертинг (минимальный включён).
