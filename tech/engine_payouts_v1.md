# Engine авто‑выплат (v1) — концепт
Очередь `payout_task` со стейт‑машиной: `queued→signing→broadcast→confirmed/failed`. Батчирование и приоритеты (pass‑through > EOD). Политики speed‑up/cancel (для ERC‑20 позже). В MVP TRC‑20 с ручным approve.
