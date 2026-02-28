# 🧾 Confirm処理フロー設計（Save Draft → Confirm 前提）

## 🎯 基本思想

- `fax_job_lines` は作業領域（OCR結果 + 下書き）
- `orders / order_lines` は正式受注台帳
- Confirm時にのみ `orders / order_lines` を作成する
- 競合防止のため `updated_at` による楽観ロックを行う
- Confirmは冪等にする（`orders.fax_job_id` は UNIQUE）

---

# 🧩 データ責務整理

| データ | 役割 |
|--------|------|
| `title_raw` | OCR結果（機械出力・原則変更しない） |
| `title_draft` | 人間作業中の値（途中保存可） |
| `order_lines.title` | 最終確定値（正式データ） |

---

# 🔄 Save Draft フロー

## API

POST /fax-jobs/{faxJobId}/save-draft

## 入力

- lineごとの `*_draft` フィールド
- `expected_fax_job_updated_at`

## サーバ処理

1. `SELECT fax_jobs FOR UPDATE`
2. `updated_at` を比較
   - 不一致 → `409 Conflict`
3. `fax_job_lines` を更新（draft値）
4. `fax_jobs.updated_at = now()` に更新
5. `COMMIT`

## ポイント

- Save Draft は何度でも実行可能
- draft は確定前の作業領域
- 正式データはまだ作らない

---

# ✅ Confirm フロー

## API

POST /fax-jobs/{faxJobId}/confirm

## 入力

- `customer_id`
- `desired_due_date`
- `notes`
- `expected_fax_job_updated_at`
- `actor`

---

# 🔐 Confirm トランザクション設計

## Step 0: BEGIN

---

## Step 1: fax_job ロック

```sql
SELECT *
FROM fax_jobs
WHERE id = :faxJobId
FOR UPDATE;
```

- updated_at を比較
- 不一致なら 409 Conflict

## Step 2: 二重確定防止チェック

```sql
SELECT id
FROM orders
WHERE fax_job_id = :faxJobId;
```

- 存在すればその order_id を返す（冪等設計）
- または 409 Conflict

※ orders.fax_job_id は UNIQUE 制約

## Step 3: 明細ロック

```sql
SELECT *
FROM fax_job_lines
WHERE fax_job_id = :faxJobId
ORDER BY line_no
FOR UPDATE;
```

### チェック

- 明細が存在すること
- 必須項目が揃っていること
- HELD行が含まれていないこと（MVPでは部分確定しない）

## Step 4: orders 作成（親）

```sql
INSERT INTO orders (...)
VALUES (...)
RETURNING id;
```

設定内容例：

- status = 'CONFIRMED'
- fax_job_id
- customer_id
- desired_due_date
- notes
- confirmed_by = actor
- confirmed_at = now()

## Step 5: order_lines 一括作成（子）

各 fax_job_line から変換：

```sql
title          = COALESCE(title_draft, title_raw)
author         = COALESCE(author_draft, author_raw)
publisher_text = COALESCE(publisher_text_draft, publisher_text_raw)
quantity       = COALESCE(quantity_draft, quantity_raw)
```

- fax_job_line_id を保存（トレース用）
- selected_api_lookup_log_id も保存
- route は route_draft > route_suggested

## Step 6: fax_jobs 更新

```sql
UPDATE fax_jobs
SET status = 'CONFIRMED',
    confirmed_order_id = :orderId,
    confirmed_by = :actor,
    confirmed_at = now(),
    updated_at = now()
WHERE id = :faxJobId;
```

## Step 7: audit_logs 追加（推奨）

```sql
INSERT INTO audit_logs (...)
VALUES (...);
```

例：

- entity_type = 'FAX_JOB'
- entity_id = fax_job_id
- action = 'CONFIRM'
- actor = actor
- message = 'Created order {orderId}'
- created_at = now()

## Step 8: COMMIT

# 🔒 競合対策まとめ

| 対策                         | 目的               |
| -------------------------- | ---------------- |
| `FOR UPDATE`               | 同時Confirm防止      |
| `updated_at` 比較            | 古い画面からのConfirm防止 |
| `orders.fax_job_id UNIQUE` | 二重確定物理防止         |
| 冪等設計                       | 二度押し安全化          |

# 📌 運用ルール

1. Confirmは1回のみ
2. Confirm前にSave Draftを行う
3. title_raw / title_draft は確定後も残す（精度測定用）
4. order_lines が最終的な正

# 🏁 全体フロー

```text
OCR(title_raw)
   ↓
人間編集(title_draft)
   ↓ Save Draft
Confirm
   ↓
orders / order_lines 作成（正式受注）
```
