# trade_history_requirements_v1 / trade_history_db_design_v1 整合性レビュー（v1）

## 1. 結論サマリ

- **全体整合性は概ね良好**。要件定義にある主要機能（マルチポートフォリオ、トランザクション中心設計、損益計算、監査、CSV取込）に対して、DB設計は十分に対応できる構成。  
- 一方で、実装時の不整合・運用事故を防ぐために、**トランザクション整合制約**、**TRANSFER表現の統一**、**削除要件の明確化**などを追加で明文化することを推奨。

---

## 2. 整合している点（主要）

1. **ドメイン中心思想の一致**
   - 要件: 「ポジションを直接更新せず、取引履歴から導出」
   - DB: `transactions` 正 + `position_snapshots` 派生 の設計方針で一致。

2. **マルチポートフォリオ/マルチアセット対応**
   - 要件: 複数ポートフォリオ、複数資産種別、複数通貨。
   - DB: `portfolios/accounts/assets/currencies/fx_rate_histories` により網羅。

3. **取引種別のカバレッジ**
   - 要件にある `BUY/SELL/CASH_IN/CASH_OUT/FEE/TAX/DIVIDEND/INTEREST/TRANSFER/SPLIT/MERGE` を
   - DBの `transaction_types` で定義済み。

4. **監査可能性**
   - 要件: 編集・取消の痕跡保持。
   - DB: `is_voided`, `revision_no`, `transaction_audit_logs` を用意。

5. **CSV取込要件**
   - 要件: プレビュー/エラー行表示/重複検知。
   - DB: `import_jobs` + `import_rows` + `dedupe_key` で対応方針あり。

---

## 3. 不整合・曖昧点（改善優先度つき）

## 優先度: 高

### A. `transactions` の `portfolio_id` と `account_id` の整合制約が未定義

- 現状のDDL案では、`transactions.account_id` が別ポートフォリオ配下の `accounts.id` を参照できてしまう余地がある。
- これが起きると「同一取引内で portfolio/account が不一致」というデータ破損が発生する。

**改善案**
- `accounts` に `(id, portfolio_id)` の複合ユニークを置く。
- `transactions` 側に `FOREIGN KEY (account_id, portfolio_id) REFERENCES accounts(id, portfolio_id)` を追加。

### B. TRANSFERの表現が曖昧（1行 or 2行）

- 要件上は「移動元・移動先」の入力が必要。
- DB案では `related_transaction_id` があるため2行モデルを想定しているが、明文化不足。

**改善案**
- 仕様として「TRANSFERは必ず2レコード（出庫/入庫）で記録し、相互に `related_transaction_id` を参照する」と明記。
- もしくは単一行モデルにするなら `from_account_id/to_account_id`（必要なら `from_portfolio_id/to_portfolio_id`）を追加。
- 現行案との親和性は**2レコード方式**が高い。

### C. 「削除」受入基準とDB方針の語彙ずれ

- 要件受入基準には「登録・編集・削除」が記載。
- DB案は「物理削除回避、`is_voided=true`」方針。

**改善案**
- 要件側の文言を「削除（論理削除/取消）」へ統一して誤解を防ぐ。

## 優先度: 中

### D. 取引種別ごとの必須項目制約がDBレベルで弱い

- 現状は「アプリ層 + トリガで担保」とあるが、必須ルールが曖昧なまま実装されるリスク。

**改善案**
- `CHECK` 制約または `BEFORE INSERT/UPDATE` トリガに、以下を明示:
  - `BUY/SELL`: `asset_id, quantity, unit_price` 必須
  - `CASH_IN/CASH_OUT/FEE/TAX/DIVIDEND/INTEREST`: `gross_amount, currency_code` 必須
  - `SPLIT/MERGE`: `asset_id` + 比率情報（後述E）必須

### E. SPLIT/MERGE の比率格納先がない

- 要件では SPLIT/MERGE に比率が必須。
- `transactions` には比率専用カラムがない。

**改善案**
- `transactions` に `corporate_action_ratio numeric(24,10)` を追加、または
- `transaction_metadata jsonb` を追加して `{"split_ratio": "2:1"}` などを保持。
- 集計再現性のため、**専用カラム推奨**。

### F. 検索要件に対するインデックス補強

- 要件はタグ絞り込み・資産種別絞り込みを含む。
- `transaction_tags` と `assets.asset_type` での検索を想定した複合インデックスが未記載。

**改善案**
- `transaction_tags(tag_id, transaction_id)`
- `assets(asset_type, id)`
- 必要に応じ `transactions(portfolio_id, asset_id, trade_date DESC)` を明示。

## 優先度: 低

### G. 金額符号ルールの統一方針を明示

- 現状 `gross_amount >= 0` のチェックがあるが、売却・出金の表現を「typeで方向管理」する前提を仕様に明記したほうが実装が安定。

**改善案**
- 「金額・数量は非負、増減方向は `transaction_type` で決定」を仕様化。

### H. ポートフォリオ別原価計算方式の導入タイミング

- DB案では将来追加可能として `cost_basis_method` が示唆されている。
- MVP-1でも平均法を使うため、初期導入しておくと移行コストが下がる。

**改善案**
- `portfolios.cost_basis_method default 'average'` をv1から追加。

---

## 4. 推奨アクション（最短）

1. DB設計書に **A/B/C/D/E** を追記（必須）。
2. 要件定義書の受入基準「削除」を **論理削除/取消** と明記。
3. マイグレーション起票時に、整合制約（複合FK）と取引種別バリデーションを最優先タスク化。

---

## 5. 判定

- **整合性判定: B+（実装可能、ただし高優先度の明文化不足あり）**
- 上記改善を反映すれば、要件→DB→実装のトレーサビリティがかなり高まり、後戻りコストを抑制できる。
