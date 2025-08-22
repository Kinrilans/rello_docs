# Основная спецификация v2 (с EOD‑неттингом)

Эта версия объединяет и **заменяет** предыдущие черновики по функциям, флоу, БД и правилам — с учётом новой функции **доверительных пар** и **EOD‑неттинга** (для аккаунтов с депозитом). Старые файлы оставлены в истории как архив.

---

## 1) Архитектура ботов (hub‑and‑spoke)
- **@ClubGateBot — Шлюз/Клуб.** Инвайты, профили, реквизиты (банк/крипто), каталог партнёров, Inbox, **доверительные пары** (/trust_pairs, /trust_partner), справка/правила.
- **@DealDeskBot — Офферы и сделки.** Лента, отклики, приватные сделки, подтверждения, базовые споры, **переключатель “Неттинг в EOD”**, `/settle_now`.
- **@TreasuryBot — Казначейка.** Депозиты, лимиты, входящие для pass‑through, авто‑выплаты, журнал выплат, **EOD‑расчёты** (/settlements, /settlement_rules).
- **@ArbiterDeskBot — Арбитраж.** Очередь споров, доказательства, решения, удержания, **trust‑session disputes**.
- **@IntegrationsBot (опц.).** API‑ключи, webhooks, deeplink‑генерация.

---

## 2) Каталог функций и команд (обновлено)

### @ClubGateBot
- `/start`, `/invite`, `/members`, `/company`, `/my_profile`, `/accounts`, `/wallets`, `/partners`, `/partner <id>`, `/inbox`, `/help`, `/rules`, `/faq`.
- **Новое:** `/trust_pairs` — список/управление; `/trust_partner <id>` — карточка пары (токены/сети, дневной лимит, TZ, cut‑off, резерв %). Доступ — Company Admin, **только при активном депозите** у обеих сторон.

### @DealDeskBot
- `/browse_offers`, `/new_offer`, `/offer <id>`, `/watchlist`, `/deal <id>`, подтверждения отправки/получения, продление дедлайна, отмена, `/dispute`.
- **Новое:** переключатель **«Неттинг в EOD»** (по умолчанию включён между доверительными партнёрами), виджет дневной позиции (X% лимита; ETA до cut‑off), кнопка **/settle_now <partner>**.

### @TreasuryBot
- `/deposit`, `/withdraw_deposit`, `/limits`, `/tiers`, `/pay_in`, `/pay_out`, `/payouts`.
- **Новое:** `/settlements` (Сегодня/Вчера/Архив; пары, net‑позиции, статусы), `/settlement_rules` (батчинг, caps, min‑confirmations, резерв), нотификации EOD (60/15/5 мин).

### @ArbiterDeskBot
- `/cases`, `/case <id>`, `/evidence`, `/resolve`, `/appeal`.
- **Новое:** фильтр/тип **trust‑session disputes**; действия: корректировка ledger, удержание/компенсация из депозита, перенос остатка T+1, штраф 0.5–1.5% при вине плательщика.

---

## 3) Диалоговые флоу (сжатая версия, полные — в отдельных файлах)

### DealDesk: /browse_offers → deal (pass‑through)
1) Выбор оффера → приватная сделка → чек‑лист (сеть, токен, min‑confirmations, дедлайн).  
2) Отправка крипто на кошелёк платформы → **auto‑payout** при подтверждениях → обе стороны отмечают получение → `completed`.  
3) Ошибки: неверная сеть, превышен cap — батчевание.

### DealDesk: /new_offer (deposit‑mode) + EOD‑неттинг
1) Создание оффера с **режимом deposit‑mode** и галкой **«Неттинг в EOD»** (активна по умолчанию, если есть доверительная пара).  
2) В треде сделки — запись в `trust_ledger` вместо авто‑выплаты; виджет «Сегодняшняя позиция A↔B».  
3) `/settle_now` — внеплановый расчёт (по запросу) → уходит в казначейку.

### ClubGate: /trust_partner
1) Форма: токены/сети (USDT/USDC; ERC‑20/TRC‑20), дневной лимит (USD экв.), TZ, **cut‑off** (например, 18:00), резерв % (опц.).  
2) Запрос партнёру → Принять/Отклонить → бейдж «Доверительный партнёр».

### Treasury: /settlements (EOD)
1) В момент cut‑off TZ пары: `session open → closed`, считается net.  
2) Создаётся `eod_settlement` → payout из hot‑кошелька (батчинг, с cap).  
3) Недостаточно ликвидности — частично; остаток → T+1; алерты.  
4) Экран: Сегодня/Вчера/Архив + CSV.

### Arbiter: trust‑session dispute
1) Основания: неверный ledger, спор суммы, неисполнение EOD.  
2) Действия: запрос материалов, корректировка записей, удержание/компенсация, штраф 0.5–1.5%, перенос на T+1.

---

## 4) Таблица депозитных tier’ов (обновлено)
- Tier’ы S/M/L/XL и `k_base` — **без изменений по цифрам**.  
- **Доступ к доверительным парам/EOD только при активном депозите** у обеих компаний.  
- **Резерв под доверительные пары:** рекомендуется `10–20%` от дневного лимита пары, удерживается как `hold` в депозите (настраивается платформой).  
- Сети/токены: USDT/USDC на ERC‑20 и TRC‑20.

---

## 5) Схема БД (добавления)
**Новые таблицы:**
- `trust_pair(id, company_a_id, company_b_id, status, tokens, networks, daily_credit_limit_usd, reserve_pct, cutoff_local_time, timezone, created_by, created_at)`
- `trust_session(id, pair_id, session_date, state, net_a_usd, net_b_usd, net_token, net_amount_token, closed_at, settled_at)`
- `trust_ledger(id, session_id, deal_id, side, token, network, amount_token, amount_usd, type, created_at)`
- `eod_settlement(id, session_id, payer_company_id, payee_company_id, token, network, amount_token, amount_usd, outgoing_tx_id, status, created_at)`

**Изменения:** `deal.is_trusted_netting bool`, `deal.trust_session_id uuid NULL`.

**Индексы:** `trust_ledger(session_id, created_at)`, нормализованная пара в `trust_pair`.

---

## 6) Правила арбитража (добавление секции)
**Trust‑session и EOD‑неттинг**
- Спор открывается в течение **24 ч** после EOD.  
- Неисполнение EOD по вине плательщика → компенсация из его депозита + **штраф 0.5–1.5%**.  
- Оспаривание ledger: обязанность предоставить материалы (чаты сделки, tx‑хэши, скрины).  
- Частичная оплата T+1 допустима при подтверждённой нехватке ликвидности платформы.

---

## 7) Операционные правила (новое)
- Планировщик по TZ пары: `EOD close → compute net → enqueue payout`.  
- Алерты за **60/15/5 мин** до cut‑off; автопауза при 90% дневного лимита пары.  
- Недостаток hot‑ликвидности → частичная выплата, остаток — приоритетный payout на T+1.

---

## 8) Нотификации (обновлено)
- @ClubGateBot: активация/пауза пары, «до cut‑off: HH:MM».  
- @DealDeskBot: текущая дневная позиция A↔B, % лимита.  
- @TreasuryBot: список сегодняшних EOD‑расчётов; статусы payout.  
- @ArbiterDeskBot: открытие/решение trust‑спора.

---

## 9) Метрики (добавлено)
- Доля сделок в EOD‑неттинге, средняя экономия на сетевых комиссиях, % частичных выплат T+1, средний net по парам, число активных доверительных пар.