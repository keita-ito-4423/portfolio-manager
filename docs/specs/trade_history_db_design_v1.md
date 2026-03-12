# トレード履歴管理ツール DB構成案（v1）

`docs/specs/trade_history_requirements_v1.md` をもとに、MVP〜将来拡張を見据えたRDB（PostgreSQL想定）の構成案を示します。

## 1. 設計方針

- **イベントソーシング寄り**：ポジションを直接更新せず、`transactions` を正として計算する。
- **監査可能性**：取引の編集・取消の痕跡を残す（論理削除 + 版管理列 + 監査ログ）。
- **マルチ資産／マルチ通貨**：資産種別・通貨・為替を正規化し、将来のAPI連携に対応。
- **再計算容易性**：スナップショットは派生データとして保持し、再生成できる構造にする。

---

## 2. エンティティ一覧

### マスタ系
- `users`
- `portfolios`
- `accounts`
- `assets`
- `currencies`
- `transaction_types`
- `tags`

### トランザクション系
- `transactions`
- `transaction_lots`（FIFO等の将来拡張用）
- `transaction_tags`
- `transaction_notes`
- `transaction_audit_logs`

### 市場データ系
- `price_histories`
- `fx_rate_histories`

### 派生・集計系
- `position_snapshots`
- `portfolio_valuation_snapshots`

### 連携・取込系
- `import_jobs`
- `import_rows`

---

## 3. テーブル定義（案）

> 型は PostgreSQL 例。金額は `numeric(24,8)`、数量は `numeric(24,10)` を基本にして桁不足を防止。

### 3.1 users

| カラム | 型 | 制約/説明 |
|---|---|---|
| id | uuid | PK |
| email | text | UNIQUE, NOT NULL |
| display_name | text |  |
| timezone | text | NOT NULL, default `Asia/Tokyo` |
| created_at | timestamptz | NOT NULL |
| updated_at | timestamptz | NOT NULL |

### 3.2 portfolios

| カラム | 型 | 制約/説明 |
|---|---|---|
| id | uuid | PK |
| user_id | uuid | FK -> users.id, NOT NULL |
| name | text | NOT NULL |
| base_currency_code | char(3) | FK -> currencies.code, NOT NULL |
| display_order | int | NOT NULL default 0 |
| is_archived | boolean | NOT NULL default false |
| created_at | timestamptz | NOT NULL |
| updated_at | timestamptz | NOT NULL |

**Index**
- `(user_id, is_archived, display_order)`

### 3.3 accounts

| カラム | 型 | 制約/説明 |
|---|---|---|
| id | uuid | PK |
| portfolio_id | uuid | FK -> portfolios.id, NOT NULL |
| account_type | text | `brokerage/exchange/wallet/cash/other` |
| name | text | NOT NULL |
| institution_name | text |  |
| external_ref | text | API連携時の口座ID |
| is_active | boolean | NOT NULL default true |
| created_at | timestamptz | NOT NULL |
| updated_at | timestamptz | NOT NULL |

**Index**
- `(portfolio_id, is_active)`

### 3.4 currencies

| カラム | 型 | 制約/説明 |
|---|---|---|
| code | char(3) | PK (JPY, USD など) |
| name | text | NOT NULL |
| decimals | smallint | NOT NULL default 2 |

### 3.5 assets

| カラム | 型 | 制約/説明 |
|---|---|---|
| id | uuid | PK |
| asset_type | text | `stock/etf/fund/crypto/commodity/cash/other` |
| symbol | text | NOT NULL |
| isin | text | nullable |
| exchange_code | text | nullable |
| display_name | text | NOT NULL |
| quote_currency_code | char(3) | FK -> currencies.code |
| price_source_type | text | `manual/api/none` |
| metadata | jsonb | 拡張情報（銘柄属性など） |
| is_active | boolean | NOT NULL default true |
| created_at | timestamptz | NOT NULL |
| updated_at | timestamptz | NOT NULL |

**Unique候補**
- `(symbol, exchange_code)`
- ISINがある場合は `isin` UNIQUE（部分ユニーク）

### 3.6 transaction_types

| カラム | 型 | 制約/説明 |
|---|---|---|
| code | text | PK (`BUY`,`SELL`,`CASH_IN`,`CASH_OUT`,`FEE`,`TAX`,`DIVIDEND`,`INTEREST`,`TRANSFER`,`SPLIT`,`MERGE`) |
| category | text | `trade/cashflow/corporate_action/cost/income` |
| affects_position | boolean | NOT NULL |
| affects_cash | boolean | NOT NULL |

### 3.7 transactions

最重要テーブル。要件定義 7.2 を包含。

| カラム | 型 | 制約/説明 |
|---|---|---|
| id | uuid | PK |
| portfolio_id | uuid | FK -> portfolios.id, NOT NULL |
| account_id | uuid | FK -> accounts.id, nullable（ポートフォリオ直下の現金移動等） |
| asset_id | uuid | FK -> assets.id, nullable（純粋入出金なら不要） |
| transaction_type | text | FK -> transaction_types.code, NOT NULL |
| trade_date | date | NOT NULL |
| settlement_date | date | nullable |
| quantity | numeric(24,10) | nullable（入出金はNULL可） |
| unit_price | numeric(24,8) | nullable |
| gross_amount | numeric(24,8) | NOT NULL default 0 |
| fee_amount | numeric(24,8) | NOT NULL default 0 |
| tax_amount | numeric(24,8) | NOT NULL default 0 |
| net_amount | numeric(24,8) | NOT NULL default 0 |
| currency_code | char(3) | FK -> currencies.code, NOT NULL |
| fx_rate_to_base | numeric(24,10) | nullable |
| related_transaction_id | uuid | FK -> transactions.id（TRANSFERの相方など） |
| source_type | text | `manual/import/api` |
| source_file | text | nullable |
| external_transaction_ref | text | 連携先ID（重複排除に使用） |
| memo | text | nullable |
| is_voided | boolean | NOT NULL default false |
| revision_no | int | NOT NULL default 1 |
| created_by | uuid | FK -> users.id |
| created_at | timestamptz | NOT NULL |
| updated_at | timestamptz | NOT NULL |

**主要制約（例）**
- `CHECK (gross_amount >= 0)`
- `CHECK (fee_amount >= 0 AND tax_amount >= 0)`
- `CHECK (transaction_type IN (...定義済みコード...))`
- BUY/SELL時の `asset_id, quantity, unit_price` 必須はアプリ層 + DBトリガで担保

**Index**
- `(portfolio_id, trade_date desc, id)`
- `(account_id, trade_date desc)`
- `(asset_id, trade_date desc)`
- `(transaction_type, trade_date desc)`
- `(source_type, source_file)`
- `(external_transaction_ref)` ※NULL除外部分インデックス推奨

### 3.8 transaction_tags

| カラム | 型 | 制約/説明 |
|---|---|---|
| transaction_id | uuid | FK -> transactions.id |
| tag_id | uuid | FK -> tags.id |
| created_at | timestamptz | NOT NULL |

PK: `(transaction_id, tag_id)`

### 3.9 tags

| カラム | 型 | 制約/説明 |
|---|---|---|
| id | uuid | PK |
| user_id | uuid | FK -> users.id |
| name | text | NOT NULL |
| color | text | nullable |
| created_at | timestamptz | NOT NULL |

UNIQUE: `(user_id, name)`

### 3.10 transaction_notes

| カラム | 型 | 制約/説明 |
|---|---|---|
| id | uuid | PK |
| transaction_id | uuid | FK -> transactions.id |
| strategy_name | text | nullable |
| rationale | text | nullable |
| note_body | text | nullable |
| created_at | timestamptz | NOT NULL |
| updated_at | timestamptz | NOT NULL |

### 3.11 transaction_audit_logs

| カラム | 型 | 制約/説明 |
|---|---|---|
| id | bigserial | PK |
| transaction_id | uuid | FK -> transactions.id |
| action | text | `INSERT/UPDATE/VOID/DELETE` |
| before_data | jsonb | 変更前 |
| after_data | jsonb | 変更後 |
| changed_by | uuid | FK -> users.id |
| changed_at | timestamptz | NOT NULL |

**Index**
- `(transaction_id, changed_at desc)`

### 3.12 price_histories

| カラム | 型 | 制約/説明 |
|---|---|---|
| asset_id | uuid | FK -> assets.id |
| price_date | date | NOT NULL |
| close_price | numeric(24,8) | NOT NULL |
| currency_code | char(3) | FK -> currencies.code |
| source_type | text | `manual/api` |
| source_name | text | nullable |
| created_at | timestamptz | NOT NULL |

PK: `(asset_id, price_date, source_type)`

### 3.13 fx_rate_histories

| カラム | 型 | 制約/説明 |
|---|---|---|
| base_currency_code | char(3) | FK -> currencies.code |
| quote_currency_code | char(3) | FK -> currencies.code |
| rate_date | date | NOT NULL |
| rate | numeric(24,10) | NOT NULL |
| source_type | text | `manual/api` |
| created_at | timestamptz | NOT NULL |

PK: `(base_currency_code, quote_currency_code, rate_date, source_type)`

### 3.14 position_snapshots

| カラム | 型 | 制約/説明 |
|---|---|---|
| id | bigserial | PK |
| portfolio_id | uuid | FK -> portfolios.id |
| account_id | uuid | FK -> accounts.id |
| asset_id | uuid | FK -> assets.id |
| snapshot_date | date | NOT NULL |
| quantity | numeric(24,10) | NOT NULL |
| avg_cost | numeric(24,8) | NOT NULL |
| book_value | numeric(24,8) | NOT NULL |
| market_price | numeric(24,8) | nullable |
| market_value | numeric(24,8) | nullable |
| unrealized_pnl | numeric(24,8) | nullable |
| realized_pnl_ytd | numeric(24,8) | nullable |
| base_currency_code | char(3) | NOT NULL |
| calculated_at | timestamptz | NOT NULL |

UNIQUE: `(portfolio_id, account_id, asset_id, snapshot_date)`

### 3.15 portfolio_valuation_snapshots

| カラム | 型 | 制約/説明 |
|---|---|---|
| id | bigserial | PK |
| portfolio_id | uuid | FK -> portfolios.id |
| snapshot_date | date | NOT NULL |
| total_market_value | numeric(24,8) | NOT NULL |
| cash_value | numeric(24,8) | NOT NULL |
| invested_amount | numeric(24,8) | NOT NULL |
| realized_pnl | numeric(24,8) | NOT NULL |
| unrealized_pnl | numeric(24,8) | NOT NULL |
| base_currency_code | char(3) | NOT NULL |
| calculated_at | timestamptz | NOT NULL |

UNIQUE: `(portfolio_id, snapshot_date)`

### 3.16 import_jobs / import_rows

#### import_jobs
- `id`, `user_id`, `portfolio_id`, `source_name`, `filename`, `status`, `started_at`, `finished_at`, `summary`

#### import_rows
- `id`, `job_id`, `row_number`, `raw_data(jsonb)`, `normalized_data(jsonb)`, `validation_errors(jsonb)`, `dedupe_key`, `created_transaction_id`

**ポイント**
- 取込プレビュー、エラー行表示、重複検知（要件5.5）に対応。

---

## 4. リレーション概要

- `users 1 - n portfolios`
- `portfolios 1 - n accounts`
- `portfolios 1 - n transactions`
- `accounts 1 - n transactions`
- `assets 1 - n transactions`
- `transactions n - n tags`（`transaction_tags`）
- `assets 1 - n price_histories`
- `position_snapshots` は `transactions + price_histories + fx_rate_histories` から生成

---

## 5. 主要クエリ観点（インデックス根拠）

1. **履歴一覧検索**
   - 条件：`portfolio_id + trade_date範囲 + transaction_type + asset_id + account_id`
   - 対応：`transactions` の複合インデックス

2. **銘柄別損益**
   - 条件：`portfolio_id + asset_id + trade_date`
   - 対応：`(asset_id, trade_date)` と `portfolio_id` 組み合わせ

3. **ダッシュボード集計**
   - 日次表示は `portfolio_valuation_snapshots` を優先参照

4. **CSV重複検知**
   - `external_transaction_ref` または `dedupe_key` を使った部分ユニークインデックス

---

## 6. 計算方式拡張への備え

- MVPでは平均法を標準実装。
- FIFO対応時は `transaction_lots` を有効化し、SELL時にlot消化履歴を保持。
- `portfolios` に `cost_basis_method`（`average/fifo`）を追加可能な設計。

---

## 7. 監査・整合性方針

- `transactions` は物理削除を避け、`is_voided=true` で取消。
- 変更前後は `transaction_audit_logs` に保存。
- バッチ再計算ジョブで `position_snapshots` を再構築可能にする。
- 会計ロック（日付以前編集禁止）が必要なら `portfolio_lock_dates` テーブル追加。

---

## 8. MVP実装優先順位（DB観点）

1. 必須：`users, portfolios, accounts, currencies, assets, transactions, transaction_types`
2. 次点：`price_histories, fx_rate_histories, position_snapshots`
3. 追加：`tags, transaction_notes, import_jobs, import_rows, transaction_audit_logs`

この順で実装すると、MVP-1→MVP-2→MVP-3 の要件段階に整合しやすい。
