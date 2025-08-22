# Мониторинг и алерты (v1)

Цель: единая система наблюдаемости для клубной платформы (боты, API, payout‑engine, Trust/EOD, кошельки, вебхуки). Стэк: **Prometheus + Alertmanager**, **OpenTelemetry (traces/logs/metrics)**, **Loki/ELK** для логов, дашборды в **Grafana**, оповещения — **@OpsAdminBot + Pager**. Часовой пояс: **Europe/Warsaw**.

---

## 0) Источники данных

- **Метрики**: экспортёры сервисов (REST/API, payout‑engine, treasury, trust, arbiter), метрики кошельков/RPC, webhooks.
- **Трейсы**: trace\_id сквозной (`deal_id`, `session_id`, `settlement_id`, `company_id`, `pair_id` — как теги).
- **Логи**: структурированные JSON с полями `level, msg, ts, trace_id, entity_type, entity_id, deal_id, company_id`.
- **Синтетика**: тест‑deeplink, тест‑webhook, тест‑payout 1 USDT (по cron, dry‑run на stg/prod‑canary).

---

## 1) SLI / SLO (начальные)

| Сервис               | SLI                                                  | SLO                |
| -------------------- | ---------------------------------------------------- | ------------------ |
| Pass‑through выплаты | p95 время `incoming_tx.confirmed → payout.confirmed` | ≤ **15 мин**       |
| EOD‑расчёты          | p95 `cut‑off → eod_settlement.confirmed`             | ≤ **60 мин**       |
| Доставка вебхуков    | Успешные за 15 мин окно                              | ≥ **99%**          |
| RPC доступность      | Аптайм RPC‑пула по сети                              | ≥ **99.9%**        |
| Горячие кошельки     | Баланс токена ≥ `min_hot_balance`                    | **100% интервала** |
| Trust пары           | Доля **partial/T+1** от объёма                       | ≤ **5%/день**      |
| Споры                | p95 время решения                                    | ≤ **72 ч**         |
| Deeplink/JWT         | Успех consume                                        | ≥ **99.5%**        |
| Боты                 | Аптайм long‑poll/Webhook                             | ≥ **99.9%**        |

**Бюджет ошибок**: 28.8 мин/мес для 99.9%; для вебхуков — ≤ 0.5% недоставок/15 мин окно.

---

## 2) Дашборды (Grafana)

1. **Executive**: SLO карты, сегодня/неделя, инциденты, объёмы, % EOD partial.
2. **Treasury**: очереди payout‑engine, p95 времени, hot‑балансы token/газ, stuck‑tx, cap per‑tx попадания, batch‑профиль.
3. **Trust/EOD**: число активных пар, загрузка лимитов (p50/p95), сегодняшние net‑позиции, cut‑off ETA, partial/T+1.
4. **Deals**: поток сделок/час, конверсия оффер→сделка, отмены, дедлайны.
5. **Webhooks**: попытки/успех, latency, dead‑letter, топ endpoint’ов по ошибкам.
6. **Arbiter**: вход/закрытия, p95 до решения, доля апелляций.
7. **Infra/RPC**: аптайм, latency, ошибки, failover, газ/fee рынок (ETH), Energy/Bandwidth (TRON).
8. **Security**: 2FA ошибки, подозрительные deeplink (replay/expired), стоп‑листы срабатывания.

---

## 3) Алёрты (правила)

**Сев. уровни**: SEV1 (крит), SEV2 (высокий), SEV3 (умеренный). Роутинг: SEV1 → Pager + @OpsAdminBot; SEV2 → @OpsAdminBot + Treas/Risk; SEV3 → @OpsAdminBot.

### 3.1 Выплаты

- **SEV1**: `payout_engine_active=0` > 5 мин **ИЛИ** очередь `payout_queue_depth{priority="high"} > 100` и растёт 10 мин.
- **SEV2**: p95 `payout_latency_seconds{purpose="pass_through"} > 1800` (30 мин) 2 окна подряд.
- **SEV2**: stuck‑tx `count > 5` за 30 мин в сети.
- **SEV3**: `payout_partial_ratio{purpose="eod"} > 0.05` (день).

### 3.2 Ликвидность/кошельки

- **SEV1**: `hot_balance_token < min_hot_balance * 0.5` 15 мин.
- **SEV2**: `hot_balance_gas < gas_safety_buffer`.
- **SEV3**: rebalance overdue (заявка > 1 ч без подтверждения).

### 3.3 RPC/сети

- **SEV1**: `rpc_up=0` у primary+backup > 2 мин.
- **SEV2**: `rpc_error_rate > 5%` 5 мин.
- **SEV3**: `eth_basefee_gwei > threshold_priority` (перевести fee\_mode).

### 3.4 Trust/EOD

- **SEV2**: `trust_eod_unsettled_sessions > 0` через 90 мин после cut‑off.
- **SEV3**: `trust_pairs_at_90pct > 0` в рабочее время (пред‑алерт).

### 3.5 Webhooks

- **SEV2**: `webhook_success_rate < 0.98` за 15 мин.
- **SEV3**: dead‑letter рост `> 50` за 10 мин.

### 3.6 Боты/Deeplink

- **SEV1**: бот недоступен (webhook 5xx/poll gap > 2 мин).
- **SEV2**: `deeplink_consume_fail_rate > 1%` 10 мин.

### 3.7 Арбитраж

- **SEV3**: SLA риск — дел `under_review` > X с ETA<12ч (эскалация Arbiter Lead).

---

## 4) Примеры PromQL

```promql
# Очередь выплат high
sum(payout_queue_depth{priority="high"})

# P95 латентности EOD
histogram_quantile(0.95, sum(rate(payout_latency_bucket{purpose="eod"}[5m])) by (le))

# Успех вебхуков за окно
sum(rate(webhook_delivered_total[5m])) / sum(rate(webhook_attempted_total[5m]))

# Hot баланс ниже порога
min_over_time(hot_wallet_balance_token[15m]) < min_hot_balance

# Активные доверительные пары на 90% лимита
count(trust_pair_utilization_ratio > 0.9)
```

---

## 5) Синтетические проверки

- **Deeplink/JWT**: раз в 5 мин — выдать токен и перейти в бота (стенд prod‑canary), ожидать ответ «Ок».
- **Webhooks**: раз в 1 мин — отправка тестового события на staging‑endpoint с проверкой подписи и ACK.
- **Payout (канареечный)**: 1 USDT/неделю по каждой сети на внутренний адрес (маркировать и исключить из аналитики).

---

## 6) Алёрты → @OpsAdminBot (payload)

```json
{
  "severity": "SEV2",
  "title": "Payout latency p95 > 30m (ERC20)",
  "impact": "pass-through delays",
  "since": "2025-08-22T14:30:00+02:00",
  "metrics": {"p95_sec": 2220, "queue": 57},
  "entity": {"network": "ERC20"},
  "runbook": "Runbooks v1 §2, §10",
  "trace_id": "...",
  "actions": ["ACK", "SNOOZE_30M", "OPEN_DASHBOARD"]
}
```

Кнопки: **ACK**, **SNOOZE (15/30/60)**, **OPEN DASHBOARD**, **OPEN RUNBOOK**.

---

## 7) Сайлэнсы/окна обслуживания

- Плановые работы: создавать **silence** на Alertmanager по меткам `service=...`, `network=...` с TTL.
- Safe‑mode: автоматический silence для части правил, алёрт «включён safe‑mode» остаётся активным.

---

## 8) Теги/лейблы

Общие: `service, component, network, token, purpose, company_id, deal_id, pair_id, session_id, settlement_id, severity`.

---

## 9) Отчёты

- **Ежедневный дайджест (09:30)**: EOD итоги (partial/T+1), p95 выплат, вебхуки, инциденты.
- **Недельный отчёт надёжности**: выполнение SLO, расход error‑budget, RCA инцидентов, план улучшений.

---

## 10) Ретенции и стоимость

- Метрики: 15 мес (downsampled после 90 дней).
- Логи: 90 дней «горячие», 365 «тёплые» (архив).
- Трейсы: 14 дней полные, 90 дней — только для SEV1/2.

---

## 11) Безопасность

- Доступ к Grafana/Alertmanager по SSO + 2FA, роли (RO/Editor/Admin).
- Секреты вебхуков/ключи — не логируем; значения хешируем в telemetry.
- Алёрты не содержат PII, только идентификаторы.

---

## 12) Связанные документы

- Runbooks (v1), Engine выплат (v1), Админ‑консоль (v1), API‑контракты (v1), События/вебхуки (v1).

