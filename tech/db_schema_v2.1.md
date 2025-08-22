@@
+# DB Schema v2.1 — Rello (minor update to v2)
+
+> Этот документ **дополняет** `tech/db_schema_v2.md` и уточняет минимальные изменения для MVP.
+> Полная спецификация — в `db_schema_v2.md`; здесь — только дельты, SQL-миграция и мотивация.
+
+## Что изменилось в v2.1 (delta)
+
+1) **Новая служебная таблица `system_kv`** — хранение рабочих курсоров и простых флагов,
+   например `watcher.tron.last_block`, `safe_mode` и т.п. (ключ/значение).
+2) **Рекомендованные индексы для производительности:**
+   - `deal(initiator_company_id, state)` — быстрые списки “мои сделки”.
+   - `deal(counterparty_company_id, state)` — быстрые списки “контрагенты”.
+   - `incoming_tx(to_address, status)` — быстрый отбор входящих по адресу + стадии.
+3) **Опционально:** CHECK-ограничения на содержимое массивов в `trust_pair`:
+   - `tokens ⊆ {'USDT','USDC'}`, `networks ⊆ {'ERC20','TRC20'}`.
+4) **Примечание по циклическим FK** (`outgoing_tx` ↔ `eod_settlement`):
+   создавать таблицы **без** взаимных FK, затем добавить оба FK через `ALTER TABLE`
+   с `DEFERRABLE INITIALLY DEFERRED`. Это упрощает вставки в одной транзакции.
+
+## Мотивация
+
+- `system_kv` упрощает эксплуатацию (watcher TRC-20, флаги safe-mode) без
+  отдельной конфигурационной системы.
+- Индексы заметно ускоряют типичные запросы ботов/вебхуков.
+- CHECK-ограничения в `trust_pair` снижают риск некорректных значений.
+
+## SQL: миграция с v2 (idempotent)
+
+```sql
+-- Рекомендуется выполнить в схеме, где находятся объекты (пример: rello)
+-- SET search_path = rello, public;
+
+-- 1) Служебная KV-таблица
+CREATE TABLE IF NOT EXISTS system_kv (
+  key        text PRIMARY KEY,
+  value      jsonb NOT NULL DEFAULT '{}'::jsonb,
+  updated_at timestamptz NOT NULL DEFAULT now()
+);
+
+-- 2) Индексы по сделкам (очереди для ботов)
+CREATE INDEX IF NOT EXISTS ix_deal_initiator_state
+  ON deal(initiator_company_id, state);
+
+CREATE INDEX IF NOT EXISTS ix_deal_counterparty_state
+  ON deal(counterparty_company_id, state);
+
+-- 3) Композитный индекс по входящим (адрес + статус)
+CREATE INDEX IF NOT EXISTS ix_incoming_to_addr_status
+  ON incoming_tx(to_address, status);
+
+-- 4) CHECK на содержимое массивов (опционально, можно валидировать позже)
+ALTER TABLE IF EXISTS trust_pair
+  ADD CONSTRAINT trust_pair_tokens_allowed
+    CHECK (tokens <@ ARRAY['USDT','USDC']::text[])
+    NOT VALID;
+
+ALTER TABLE IF EXISTS trust_pair
+  ADD CONSTRAINT trust_pair_networks_allowed
+    CHECK (networks <@ ARRAY['ERC20','TRC20']::text[])
+    NOT VALID;
+
+-- Позже, при необходимости, выполнить:
+-- ALTER TABLE trust_pair VALIDATE CONSTRAINT trust_pair_tokens_allowed;
+-- ALTER TABLE trust_pair VALIDATE CONSTRAINT trust_pair_networks_allowed;
+```
+
+## Заметка по циклическим FK (пример)
+
+```sql
+-- Шаг 1: создать outgoing_tx и eod_settlement БЕЗ взаимных FK
+-- Шаг 2: затем добавить оба FK deferrable
+ALTER TABLE outgoing_tx
+  ADD CONSTRAINT fk_outgoing_eod
+  FOREIGN KEY (eod_settlement_id) REFERENCES eod_settlement(id)
+  ON UPDATE CASCADE ON DELETE SET NULL
+  DEFERRABLE INITIALLY DEFERRED;
+
+ALTER TABLE eod_settlement
+  ADD CONSTRAINT fk_eod_outgoing
+  FOREIGN KEY (outgoing_tx_id) REFERENCES outgoing_tx(id)
+  ON UPDATE CASCADE ON DELETE SET NULL
+  DEFERRABLE INITIALLY DEFERRED;
+```
+
+## Совместимость
+
+- Изменения обратимо-совместимы с v2: не трогаем существующие поля/типы,
+  добавляем только новые объекты и индексы.
+- Применение миграции безопасно для пустой и для непустой базы (все DDL — IF NOT EXISTS).
+
+## Связанные разделы
+
+- `tech/webhooks_events_v1.md` — модель событий (для watcher и treasury).
+- `tech/engine_payouts_v1.md` — очереди/приоритеты выплат (используют индексы).
+- `ops/monitoring_alerts_v1.md` — наблюдаемость; `system_kv` может хранить служебные флаги.
+```
