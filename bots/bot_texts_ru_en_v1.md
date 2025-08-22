# Словарь текстов ботов (v1) — RU/EN (Club/Deal/Treasury/Arbiter/Ops)

Формат: ключи i18n → RU/EN. Плейсхолдеры в фигурных скобках: `{amount_token}`, `{token}`, `{network}`, `{company}`, `{partner}`, `{deal_id}`, `{offer_id}`, `{session_date}`, `{cutoff}`, `{tz}`, `{tx_hash}`, `{address}`, `{reason}`, `{minutes}`, `{percent}`, `{k_factor}`, `{tier}`, `{limit_usd}`, `{remaining}`, `{id}`, `{url}`.

Гайд по стилю: коротко, по делу, без жаргона; в одном сообщении — одно действие; CTA кнопкой. Русский — «вы/ваш», английский — нейтральный. Разметка: Markdown (Telegram). Числа: разделитель пробел (10 000), две цифры после запятой где нужно.

---

## Общие (shared)

- `app.welcome`
  - RU: "Привет! Это закрытая площадка клуба. Доступ — по инвайту. Выберите действие ниже."
  - EN: "Welcome to the private club platform. Access is invite‑only. Choose an action below."
- `app.denied`
  - RU: "Доступ ограничен. Обратитесь к администратору клуба."
  - EN: "Access denied. Please contact a club admin."
- `app.maintenance`
  - RU: "⚠️ Техническое окно. Часть функций временно недоступна."
  - EN: "⚠️ Maintenance window. Some features are temporarily unavailable."
- `app.safemode`
  - RU: "🛡 Включён safe‑mode: {mode}. Новые выплаты ограничены."
  - EN: "🛡 Safe‑mode active: {mode}. New payouts are limited."
- `app.deeplink.invalid`
  - RU: "Ссылка недействительна или устарела. Запросите новую кнопку."
  - EN: "This link is invalid or expired. Please request a new button."
- `app.deeplink.aud_mismatch`
  - RU: "Эта кнопка не для этого бота. Откройте её в {bot}."
  - EN: "This button belongs to {bot}. Please open it there."
- `app.rate_limited`
  - RU: "Слишком много запросов. Повторите через {minutes} мин."
  - EN: "Too many requests. Try again in {minutes} min."
- `btn.back`
  - RU: "◀️ Назад"
  - EN: "◀️ Back"
- `btn.cancel`
  - RU: "Отмена"
  - EN: "Cancel"
- `btn.ok`
  - RU: "Ок"
  - EN: "OK"

---

## @ClubGateBot — профиль, партнёры, доверительные пары

- `club.start`
  - RU: "Вы в главном меню клуба. Компания: **{company}**. Чем займёмся?"
  - EN: "Main menu. Company: **{company}**. What do you need?"
- `club.switch_company`
  - RU: "Выберите компанию для работы."
  - EN: "Select a company to proceed."
- `club.profile.setup`
  - RU: "Заполните профиль: название юрлица, контакты, часовой пояс."
  - EN: "Complete your profile: legal name, contacts, time zone."
- `club.accounts.add`
  - RU: "Добавьте банковские реквизиты/крипто‑адреса компании. Новые адреса проходят верификацию."
  - EN: "Add company bank details/crypto addresses. New addresses require verification."
- `club.partner.browse`
  - RU: "Партнёры клуба. Нажмите, чтобы открыть профиль или предложить сделку."
  - EN: "Club partners. Tap to open a profile or propose a deal."
- `club.trust.request`
  - RU: "Запрос доверительной пары {partner}. Укажите дневной лимит (USD), cut‑off и TZ."
  - EN: "Request a trusted pair with {partner}. Set daily limit (USD), cut‑off and time zone."
- `club.trust.activated`
  - RU: "Пара **{company} ↔ {partner}** активна. Лимит: {limit\_usd} USD, cut‑off {cutoff} ({tz})."
  - EN: "Pair **{company} ↔ {partner}** is active. Limit: {limit\_usd} USD, cut‑off {cutoff} ({tz})."
- `club.trust.paused`
  - RU: "Пара приостановлена. Неттинг по EOD временно отключён."
  - EN: "Pair paused. EOD netting is temporarily disabled."

---

## @DealDeskBot — офферы, сделки, неттинг

- `deal.feed`
  - RU: "Лента офферов. Фильтр: сеть {network}, токен {token}."
  - EN: "Offers feed. Filter: network {network}, token {token}."
- `deal.offer.create`
  - RU: "Создайте оффер: сеть, токен (USDT/USDC), объём, цена/курс, дедлайн."
  - EN: "Create an offer: network, token (USDT/USDC), volume, price/rate, deadline."
- `deal.offer.posted`
  - RU: "Оффер опубликован. ID {offer\_id}. Ожидаем отклики."
  - EN: "Offer posted. ID {offer\_id}. Waiting for responses."
- `deal.offer.accepted`
  - RU: "Оффер принят. Создана сделка #{deal\_id}."
  - EN: "Offer accepted. Deal #{deal\_id} created."
- `deal.mode.choice`
  - RU: "Выберите режим: **Pass‑through** (автовыплата после входящего) или **Deposit‑mode** (прямые расчёты, депозит как гарантия)."
  - EN: "Choose mode: **Pass‑through** (auto‑payout after incoming) or **Deposit‑mode** (direct settlement, deposit as guarantee)."
- `deal.pay_in.instructions`
  - RU: "Отправьте {amount\_token} {token} в сети {network} на адрес:

```
{address}
```

После {confirmations} подтверждений будет авто‑выплата контрагенту."

- EN: "Send {amount\_token} {token} on {network} to:

```
{address}
```

After {confirmations} confirmations we auto‑pay the counterparty."

- `deal.deposit_mode.note`
  - RU: "В Deposit‑mode расчёт выполняется между сторонами. Споры — через арбитраж; депозит покрывает удержания."
  - EN: "In Deposit‑mode parties settle directly. Disputes go to arbitration; deposit covers withholdings."
- `deal.trusted.toggle_on`
  - RU: "✅ Сделки с {partner} будут копиться в EOD‑сессии (неттинг включён)."
  - EN: "✅ Deals with {partner} will accumulate in the EOD session (netting on)."
- `deal.deadline.soon`
  - RU: "⏰ Дедлайн через {minutes} мин. Нужны действия."
  - EN: "⏰ Deadline in {minutes} min. Action required."
- `deal.cancelled`
  - RU: "Сделка #{deal\_id} отменена: {reason}."
  - EN: "Deal #{deal\_id} cancelled: {reason}."

---

## @TreasuryBot — депозиты, выплаты, EOD

- `treasury.deposit.address`
  - RU: "Адрес для депозита ({token} {network}):\n\`\`\` {address}

````"
  - EN: "Deposit address ({token} {network}):\n```
{address}
```"
- `treasury.deposit.funded`
  - RU: "Депозит пополнен: {amount_token} {token} ({network}). Лимит обновлён."
  - EN: "Deposit funded: {amount_token} {token} ({network}). Limit updated."
- `treasury.withdraw.requested`
  - RU: "Запрос на вывод {amount_token} {token} принят. Окно T+N началось."
  - EN: "Withdrawal for {amount_token} {token} requested. T+N window started."
- `treasury.payout.sent`
  - RU: "Выплата отправлена: {amount_token} {token} ({network}). Tx: `{tx_hash}`."
  - EN: "Payout sent: {amount_token} {token} ({network}). Tx: `{tx_hash}`."
- `treasury.payout.confirmed`
  - RU: "Выплата подтверждена. Tx: `{tx_hash}`."
  - EN: "Payout confirmed. Tx: `{tx_hash}`."
- `treasury.address.not_allowed`
  - RU: "Адрес {address} не в allow‑list вашей компании. Добавьте и пройдите верификацию."
  - EN: "Address {address} is not in your company allow‑list. Add and verify it first."
- `treasury.eod.settlement.sent`
  - RU: "EOD по паре {partner} за {session_date}: выплата {amount_token} {token} отправлена."
  - EN: "EOD with {partner} for {session_date}: payout {amount_token} {token} sent."
- `treasury.eod.settlement.partial`
  - RU: "EOD частично: оплачено {amount_token} {token}, остаток {remaining} — **T+1** с приоритетом."
  - EN: "EOD partial: paid {amount_token} {token}, remaining {remaining} — **T+1** with priority."

---

## @ArbiterDeskBot — споры/решения
- `arbiter.case.opened`
  - RU: "Открыт спор #{id} по сделке #{deal_id}. Укажите причину и приложите материалы."
  - EN: "Dispute #{id} opened for deal #{deal_id}. Provide a reason and attach evidence."
- `arbiter.case.request_evidence`
  - RU: "Требуются материалы: {reason}. Срок — {hours} ч."
  - EN: "Evidence required: {reason}. Due in {hours} h."
- `arbiter.case.decision`
  - RU: "Решение по спору #{id}: **{decision}**. Сумма: {amount_token} {token}."
  - EN: "Decision for case #{id}: **{decision}**. Amount: {amount_token} {token}."
- `arbiter.case.penalty_hold`
  - RU: "На депозите удержано {amount_token} {token} до исполнения решения."
  - EN: "Deposit hold of {amount_token} {token} applied until resolution."

---

## @OpsAdminBot — алерты и действия
- `ops.alert.header`
  - RU: "[{severity}] {title}\nС {since}. Детали: {metrics}."
  - EN: "[{severity}] {title}\nSince {since}. Details: {metrics}."
- `ops.alert.buttons`
  - RU: "ACK / SNOOZE / Дашборд / Ранбук"
  - EN: "ACK / SNOOZE / Dashboard / Runbook"
- `ops.safe_mode.on`
  - RU: "Safe‑mode включён: {mode}."
  - EN: "Safe‑mode enabled: {mode}."
- `ops.safe_mode.off`
  - RU: "Safe‑mode выключен."
  - EN: "Safe‑mode disabled."
- `ops.webhooks.degraded`
  - RU: "Доставка вебхуков деградировала {from}–{to}. Выполняем ретраи и готовим реплей."
  - EN: "Webhook delivery degraded {from}–{to}. Retries in progress, replay queued."

---

## Кнопки (keyboard)
- `kb.main`
  - RU: ["Сделки", "Партнёры", "Доверительные пары", "Реквизиты", "Профиль"]
  - EN: ["Deals", "Partners", "Trusted pairs", "Accounts", "Profile"]
- `kb.deal.actions`
  - RU: ["Создать оффер", "Лента офферов", "Мои сделки", "Включить неттинг"]
  - EN: ["Create offer", "Offers feed", "My deals", "Enable netting"]
- `kb.treasury`
  - RU: ["Адрес депозита", "Вывод депозита", "Мои выплаты", "EOD расчёты"]
  - EN: ["Deposit address", "Withdraw deposit", "My payouts", "EOD settlements"]
- `kb.arbiter`
  - RU: ["Открыть спор", "Мои споры", "Загрузить материалы"]
  - EN: ["Open dispute", "My cases", "Upload evidence"]

---

## Ошибки/валидации
- `err.required`
  - RU: "Введите значение."
  - EN: "Please enter a value."
- `err.format`
  - RU: "Неверный формат. Проверьте и повторите."
  - EN: "Invalid format. Please check and retry."
- `err.limit_exceeded`
  - RU: "Превышен лимит: {reason}."
  - EN: "Limit exceeded: {reason}."
- `err.address_invalid`
  - RU: "Адрес {address} неверен для сети {network}."
  - EN: "Address {address} is invalid for {network}."
- `err.not_allowed`
  - RU: "Недостаточно прав для этого действия."
  - EN: "You don’t have permission to do this."
- `err.try_later`
  - RU: "Не получилось. Попробуйте позже."
  - EN: "Something went wrong. Try again later."

---

## Хелпы/подсказки
- `help.netting`
  - RU: "Неттинг EOD суммирует сделки дня с доверительным партнёром и делает один расчёт в конце дня. Нужен активный депозит у обеих сторон."
  - EN: "EOD netting sums the day’s deals with a trusted partner into one payout at day end. Requires active deposits on both sides."
- `help.deposit`
  - RU: "Депозит даёт лимит на сделки: `депозит × k‑фактор`. Удержания покрывают споры и просрочки."
  - EN: "Deposit grants deal limit: `deposit × k‑factor`. Holds cover disputes and delays."
- `help.pass_through`
  - RU: "В pass‑through вы отправляете стейблы на адрес Платформы; после подтверждений мы автоматически платим контрагенту."
  - EN: "In pass‑through you send stablecoins to the Platform; after confirmations we automatically pay the counterparty."

---

## Примечания по локализации
- Не переводить названия сетей/токенов и статусов (`pass‑through`, `deposit‑mode`, `EOD`).  
- Для дат использовать формат: RU — `DD.MM.YYYY HH:mm (TZ)`, EN — `YYYY‑MM‑DD HH:mm (TZ)`.  
- Все суммы показывать в токене и (при необходимости) в USD: `10 000 USDT (~10 000 USD)`.

````
