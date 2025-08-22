# Engine авто‑выплат и EOD‑батчинг (v1)

Цель: единый сервис, который **порождает, планирует, подписывает и отправляет** выплаты по трём источникам: (1) pass‑through сделки, (2) EOD‑расчёты доверительных пар (netting), (3) ручные/арбитражные выплаты. Токены: **USDT/USDC**. Сети: **ERC‑20/TRC‑20**.

---

## 0) Объекты и роли

- **payout\_task** — элементарная выплата: `company_id → to_address, network, token, amount_token, purpose, source_ref`.
- **payout\_batch** — набор `payout_task` для одной сети/токена, собранный с учётом cap per‑tx и приоритета.
- **wallet (hot)** — отправляющий кошелёк сети. Несколько на сеть (ротация/перегрев).
- **priority** — `high` (pass‑through), `medium` (EOD), `normal` (manual), `arbiter` (по решению).
- **rate‑limit** — глобальные и на кошелёк: rps, max outstanding.

---

## 1) Источники задач

- **Pass‑through**: триггер `incoming_tx.confirmed(deal_id)` → создать `payout_task` на сумму сделки → очередь `high`.
- **EOD‑неттинг**: триггер `trust_session.closed(session_id)` → рассчитать net → создать `payout_task` на кредитора → очередь `medium`.
- **Ручные/арбитраж**: `POST /pay-out`, `arbiter.decision.posted` (refund/penalty/split) → очередь по умолчанию `normal`/`arbiter`.

Для всех `POST` денежных действий — **Idempotency‑Key** (365 дней).

---

## 2) Машина состояний выплат

`queued → funding_check → signing → broadcast → confirmed | partial | failed`

- **funding\_check**: проверка доступного баланса hot‑кошелька, сетевых cap’ов, allow‑list адреса.
- **signing**: создание/подпись tx (HSM/подписант), версионирование nonce.
- **broadcast**: отправка, сохранение `tx_hash`, трекинг подтверждений.
- **confirmed**: фиксация, закрытие задачи, вебхук `payout.sent/confirmed`.
- **partial**: когда сумма > cap per‑tx — см. батчевание.
- **failed**: с кодом причины (insufficient\_funds, bad\_nonce, fee\_too\_low, invalid\_addr, signer\_unavailable, network\_error).

---

## 3) Батчевание и cap per‑tx

**Задача:** соблюсти платформенные и сетевые ограничения, дробя сумму на несколько tx.

Алгоритм (упрощено):

1. `chunk = min(amount_token, cap_per_tx(network, token))`.
2. Добавить `payout_task_chunk` в `payout_batch` до полного покрытия суммы.
3. Между отправками — **паузу** `batch_pause_ms(network)`.
4. Для ERC‑20 учесть комиссии газа из горячего баланса (в ETH); для TRC‑20 — **энергию/TRX**.

**Приоритеты:** внутри сети сначала `high`, затем `arbiter`, затем `medium`, затем `normal`.

---

## 4) Подтверждения и min‑confirmations

- `min_conf_out(network)` — конфигурация (напр., ERC‑20: 6; TRC‑20: 20).
- Счётчик подтверждений → upon reach → статус `confirmed`, вебхук.
- Failover: отсутствие подтверждений > `conf_timeout_min` → эскалация (см. инциденты).

---

## 5) Оценка комиссий

- **ERC‑20**: EIP‑1559 — оценка `maxFeePerGas`/`maxPriorityFee`; `gas_limit` для USDT/USDC transfer (исторический усреднённый + буфер).
- **TRC‑20**: контроль **Energy/Bandwidth**; при нехватке — докупка/пополнение TRX или перераспределение на другой hot‑кошелёк.
- Политика `fee_mode`: `economy|balanced|priority` (переключаемая админом/по приоритету задачи).

---

## 6) Горячие/холодные кошельки и ребаланс

- **min\_hot\_balance(network, token)** — пороги; при падении ниже — ставим задачу **rebalance** из cold (требует ручного подтверждения/мультиподписей).
- **safety\_buffer** для газа (ETH/TRX) — минимум X, ниже — блокируем новые выплаты, отправляем алерт.
- Политика *drain*: при перегреве сети/алертах можно временно остановить выплаты или перенаправить на другой hot.

---

## 7) Контроли риска и валидации

- **Allow‑list адресов** (из `/wallets` компании). Новый адрес — только после верификации в шлюзе.
- Внутренние **caps**: дневной cap авто‑выплат, cap на одну транзакцию, cap на адрес получателя.
- Для ручных выплат — **2FA** инициатора + подтверждение **Approver** по порогам tier.
- Защита от дублирования: идемпотентность + уникальность `(network, tx_hash)`.

---

## 8) Идемпотентность и дедупликация

- Ключ: `Idempotency-Key` + `company_id` + `purpose`.
- Повторный `POST` возвращает **тот же** объект/статус.
- Храним последнюю попытку/ответ на протяжении 365 дней.

---

## 9) Интеграция с БД и событиями

- `payout_task`/`payout_batch` — таблицы очередей.
- Запись в `deposit_ledger` (при штрафах/компенсациях/арбитраже).
- Вебхуки: `payout.sent`, `payout.confirmed`, `trust.eod_settlement.*`.

---

## 10) Инциденты и обработка ошибок

- **insufficient\_funds**: ставим `partial`/`queued_T+1`, алерт казначейству (@TreasuryBot Inbox).
- **bad\_nonce**: автоматический **resync** nonce и ретрай.
- **fee\_too\_low**: поднять `fee_mode` до `priority`, ретрай.
- **invalid\_addr**: отказ (final), уведомить инициатора.
- **network\_error**: экспоненциальный бэкофф + свитч на резервный RPC.
- **stuck\_tx** (> `stuck_timeout_min`): `speed_up`/`cancel` (ERC‑20) по политике.

---

## 11) Конфигурация (по умолчанию, управляется из админки)

- `cap_per_tx`: ERC‑20 = 250 000 USDT экв., TRC‑20 = 200 000.
- `min_conf_out`: ERC‑20 = 6, TRC‑20 = 20.
- `batch_pause_ms`: ERC‑20 = 3000, TRC‑20 = 1500.
- `fee_mode_default = balanced` (pass‑through: priority; EOD: balanced; manual: economy/balanced).
- `min_hot_balance`: на 3 средних выплаты дня + буфер газа.

---

## 12) Метрики и алерты

- **Очереди:** глубина по приоритетам/сетям, время в очереди, % ретраев.
- **Успех:** доля `confirmed`, средняя латентность до `confirmed`.
- **Комиссии:** медиана fee/tx по сети и по приоритетам, доля `speed_up`/`cancel`.
- **Ликвидность:** hot\_balance (token/газ), количество `partial/T+1` выплат.
- **Качество:** частота `invalid_addr`, `bad_nonce`, `network_error`.

Алерты: низкий hot\_balance, рост очереди/латентности, % ошибок > порога, stuck\_tx > порога, выключенные вебхуки.

---

## 13) Псевдокод: pass‑through

```pseudo
on incoming_tx.confirmed(deal_id):
  deal = load(deal_id)
  if deal.mode != pass_through or deal.payout_done: return
  task = make_payout_task(deal.company_to, deal.token, deal.network, deal.amount, purpose="pass_through", ref=deal_id)
  enqueue(task, priority=high)
```

## 14) Псевдокод: EOD‑неттинг

```pseudo
on trust_session.closed(session_id):
  net = compute_net(session_id)
  if net.amount_token == 0: return
  (payer, payee) = (net.payer_company_id, net.payee_company_id)
  task = make_payout_task(payee, net.token, chosen_network, net.amount_token, purpose="eod", ref=session_id)
  enqueue(task, priority=medium)
```

---

## 15) Безопасность и аудит

- Все переходы состояния логируются (`audit_log`), включая параметры fee/nonce/tx\_hash.
- Доступ к конфигурации — только из **админ‑консоли** с 2FA и ролями.
- Журналы подписи хранят ссылку на подписанта/ключ (без раскрытия ключа).

---

## 16) Связанные документы

- **API‑контракты (v1)** — разделы Treasury и Trust/EOD.
- **События/вебхуки (v1)** — схемы `payout.*`, `trust.*`.
- **Таблица tier’ов (v2)** — cap’ы/мин‑конфирмы.
- **Runbooks** — инциденты сети, ликвидность hot/cold, stuck tx, T+1.
