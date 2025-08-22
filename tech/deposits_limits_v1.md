# Модуль депозитов и лимитов (v1)

Цель: единые правила учёта депозитов компаний и расчёта лимитов (k‑фактор), экспозиции, резервов под доверительные пары и удержаний при спорах. Токены — **USDT/USDC**; сети — **ERC‑20/TRC‑20**. Все суммы в расчётах — в **USD экв.** (1:1 к стейблам), хранение — **в токенах и USD**.

---

## 0) Сущности и определения

- **deposit** — агрегированный баланс компании на кошельках Платформы (по токенам/сетям) с учётом pending‑операций.
- **deposit\_ledger** — бухгалтерская книга движений депозита (строки атомарны, неизменяемы; исправления — отдельной строкой `adjustment`).
- **exposure\_open** — текущая открытая экспозиция компании (обычные сделки + trust‑сессии до EOD‑расчёта + споры).
- **holds/reserves** — удержания под споры и доверительные пары; не участвуют в расчёте доступного лимита.
- **k\_base** — коэффициент лимита по tier (см. Таблицу tier’ов v2).

---

## 1) Ключевые формулы

**1.1 Доступный депозит**

```
deposit_available = deposit_balance
                   - hold_total
                   - reserve_total
                   - withdraw_pending
```

**1.2 Лимит компании**

```
company_limit = deposit_available × k_base
```

**1.3 Cap’ы**

- `cap_per_deal` = процент от `company_limit` (по tier).
- `cap_auto_payout_daily` = доля `company_limit` (pass‑through).
- `cap_open_exposure` = доля `company_limit`.

**1.4 Лимит доверительной пары (рекомендация по tier)**

```
trust_pair_daily_limit ≤ min(company_limit_A, company_limit_B, cap_platform)
и ≤ «лимит пары по tier» (см. табл.)
```

---

## 2) Операции и состояния депозита

**Состояния вывода:** `requested → pending_T+N → sent → confirmed|failed`.

**Операции (в ledger):**

- `deposit_fund(+token)` — поступление депозита (после min‑confirmations).
- `withdraw_request(-token)` — создание заявки на вывод (блокирует сумму).
- `withdraw_sent(-token)` — фактическая отправка.
- `hold_open(-usd)` — удержание под спор/нарушение.
- `hold_release(+usd)` — снятие удержания.
- `penalty(-usd)` — штраф в пользу платформы/контрагента.
- `compensation(-usd)` / `refund(+usd)` — компенсация/возврат.
- `reserve_trust(-usd)` — резерв под доверительную пару (процент от лимита пары).
- `reserve_release(+usd)` — снятие резерва.
- `adjustment(±usd)` — бухгалтерская корректировка (с ссылкой на причину/решение).
- `dispute_freeze(-usd)` — временная заморозка до решения (альтернатива hold\_open для массовых кейсов).

> Примечание: строки `..._usd` хранят эквивалент в USD + разбивку по токенам/сетям, если проводка в токенах.

---

## 3) Экспозиция и связь со сделками

**exposure\_open** учитывает:

- (А) **pass‑through**: суммы входящих, по которым ещё не выполнен `auto_payout_sent`;
- (Б) **deposit‑mode**: спорные сделки с выставленным `hold_open` (или `dispute_freeze`);
- (В) **trust‑session**: текущий дневной net до EOD (через `reserve_trust` и/или оперативный расчёт из ledger);
- (Г) **арбитражные решения** до фактической выплаты.

Экспозиция **должна быть нулевой**, чтобы начать окно T+N на вывод депозита.

---

## 4) Лимиты по ролям и внутренние ограничения

- **Company Admin** может задать **внутренние лимиты** (per user/role): `daily_limit_usd`, `single_tx_limit_usd` — применяются в ботах и API (422 при превышении).
- **Approver** подтверждает выплаты/выводы **выше порога tier**.
- **Trader** ограничен лимитами роли и компании; доступ к EOD — только при активной доверительной паре.

---

## 5) Политики hold/reserve

- **Hold при споре**: `hold = min(сумма сделки/ущерба, deposit_available нарушителя)`.
- **Reserve для доверительной пары**: платформа удерживает `reserve_pct × trust_pair_daily_limit` (см. tier), проводка `reserve_trust` при активации пары; `reserve_release` при паузе/деактивации.
- **Автопауза неттинга**: при достижении **90%** дневного лимита пары новые неттинговые сделки блокируются.

---

## 6) Правила вывода депозита (T+N)

1. Вывод возможен только при `exposure_open = 0` и отсутствии активных `dispute`/`trust_session` (не settled).
2. Начало окна **T+N** фиксируется в момент `withdraw_request`.
3. Для M/L/XL — требуется подтверждение **Approver**.
4. В окне T+N новые проводки по депозиту запрещены, кроме `hold_open` по спору (что может отменить/сдвинуть вывод).

---

## 7) Согласованность и округления

- Учёт ведётся в минимальных долях токена (6–8 знаков) и в USD с точностью до **2 знаков** (банковское округление).
- Конверсия стейблов → USD = 1:1; небольшой дрейф (комиссии/кросс‑сеть) учитывается в `fee_adjustment` (в ledger).

---

## 8) События/вебхуки (см. «События v1»)

- `deposit.funded`, `deposit.withdraw.requested/sent`, `trust.pair.activated/paused`, `trust.eod_settlement.*`, `arbiter.decision.posted` — все они изменяют `deposit_available` и/или `company_limit`.

---

## 9) База данных (минимальный набор)

- `deposit(company_id, token, network, balance_token, balance_usd, updated_at)`
- `deposit_ledger(id, company_id, type, sign, token, network, amount_token, amount_usd, ref_type, ref_id, created_at, created_by)`
- `company_limit_snapshot(company_id, limit_usd, deposit_available_usd, k_base, cap_per_deal_usd, cap_auto_payout_daily_usd, cap_open_exposure_usd, created_at)`
- `internal_limits(company_id, role, daily_limit_usd, single_tx_limit_usd, created_at, created_by)`

Индексы: по `(company_id, created_at)` в `deposit_ledger`; по `ref_type/ref_id`.

---

## 10) API (перекрёстные ссылки)

- `GET /deposits`, `POST /deposits/withdraw` — пополнение/вывод.
- `GET /limits` — вернуть лимиты и snapshot.
- `GET /trust/pairs`, `POST /trust/pairs` — резервы под пары.
- Вебхуки: `deposit.*`, `trust.*`, `arbiter.*`.

---

## 11) Псевдокод

**Расчёт лимита при событии ledger**

```pseudo
on deposit_ledger.append(row):
  deposit_available = balance - holds - reserves - withdraw_pending
  company_limit = deposit_available * k_base
  snapshot(company_limit, caps)
  publish_event('company.limit.updated', ...)
```

**Проверка лимита при создании/принятии сделки**

```pseudo
if deal.amount_usd > cap_per_deal or exposure_open + deal.amount_usd > cap_open_exposure:
  reject(422, 'limit_exceeded')
```

**Активация доверительной пары**

```pseudo
reserve = pair.daily_credit_limit_usd * reserve_pct
append_ledger(company_id, 'reserve_trust', -reserve, ref=pair_id)
```

---

## 12) Крайние случаи

- Расхождение суммы по сети (комиссии/ошибка «ровной суммы») → ручная корректировка `adjustment` + запись `fee_adjustment`.
- Спор открылся в окно T+N → `hold_open` отменяет вывод до решения.
- Смена tier/k\_base админом → пересчёт лимитов на ближайшем событии ledger (и/или периодический крон).
- Перевод части депозитов между токенами/сетями (ребаланс) — через `adjustment` с комментарием и связанный `outgoing_tx`/`incoming_tx`.

---

## 13) Метрики и алерты

- Доля `deposit_available` к `deposit_balance`, частота `hold_open`, средний `company_limit` по портфелю.
- Отказы по `limit_exceeded`, доля блокировок по внутренним лимитам.
- Срабатывания автопаузы доверительных пар (>=90%).
- Очередь выводов (T+N) и случаи отмены вывода из‑за споров.

---

## 14) Безопасность и аудит

- Все операции с депозитом требуют ролей/2FA по порогам tier.
- `deposit_ledger` — неизменяемый; исправления только через `adjustment`/`correction`.
- В `audit_log` пишем инициатора, IP/user‑agent (если есть), ссылочные id.
- Экспорт журнала — только для админов/аудита по запросу.

