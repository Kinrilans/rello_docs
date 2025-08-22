# Каталог функций и команд (v2) — по ботам
## ClubGateBot
- Профиль компании, реквизиты (банковские/крипто), allow‑list адресов.
- Партнёры (каталог), запрос и активация доверительных пар (лимит USD, cut‑off, TZ).

## DealDeskBot
- Лента офферов; создание/поиск/принятие оффера.
- Создание сделки (pass‑through или доверительный EOD).
- Уведомления о дедлайнах, статусы.

## TreasuryBot
- Депозит: адрес, пополнение/вывод (T+N позже).
- Выплаты: список `payout_task`, approve/send, статусы/tx‑hash.
- EOD: список нетто‑расчётов, ручной запуск.

## ArbiterDeskBot
- Открыть спор; загрузка материалов; шаблонные решения (refund/penalty/hold_release).
- Удержания/проводки в депозитный ledger.

## OpsAdminBot
- Алерты (SEV1/2), быстрые кнопки (ACK/SNOOZE/Runbook/Dashboard).
- Тогглы safe‑mode (EOD off, freeze payouts/withdraw).
