# 06. 命名規則：チームで読めるテーブル・カラム名の付け方

---

## 1. 命名規則が重要な理由

- DBのテーブル・カラム名は「コードより長生きする」
- 後から変えると影響範囲が広い（アプリコード・クエリ・ORMマッピング・API仕様全部）
- チームメンバーが読んだとき「何が入っているか」が即座にわかるべき

---

## 2. 基本ルール

### snake_case を使う

```sql
-- ✅ Good
user_id
created_at
order_total_amount

-- ❌ Bad
userId          -- キャメルケース（SQLは大文字小文字を区別しないDBが多い）
OrderTotalAmount
CREATED_AT      -- 全大文字（古いスタイル）
```

### 英語を使う（日本語・ローマ字は禁止）

```sql
-- ✅ Good
users, products, order_items

-- ❌ Bad
ユーザー                  -- 日本語
tsuika_nichi             -- ローマ字
```

### テーブル名は**複数形**

```sql
-- ✅ Good
users, products, orders, categories, order_items

-- ❌ Bad（賛否あるが複数形が主流）
user, product, order
```

---

## 3. テーブル名の命名

### 基本パターン

| パターン | 例 | 用途 |
|---------|-----|------|
| 単純な名詞（複数形） | `users`, `products` | エンティティ本体 |
| `entity_infos` | `user_profiles` | エンティティの詳細情報 |
| `entity_entity` | `user_roles`, `product_categories` | 中間テーブル |
| `entity_histories` | `order_histories` | 履歴テーブル |
| `m_xxxx` or `mst_xxxx` | `m_categories`, `mst_prefectures` | マスタテーブル（チームによる） |

### 名前から内容が明確にわかるようにする

```sql
-- ✅ 内容が明確
user_addresses          -- ユーザーの住所
product_images          -- 商品の画像
order_status_histories  -- 注文ステータスの変更履歴

-- ❌ 何が入っているか不明
data
info
temp
records
```

---

## 4. カラム名の命名

### ID カラム

```sql
-- テーブル内のPK → id
id INT UNSIGNED PRIMARY KEY

-- 他テーブルの参照 → {テーブル単数形}_id
user_id INT UNSIGNED
order_id INT UNSIGNED
product_id INT UNSIGNED
```

### 真偽値カラム

```sql
-- is_ または has_ または can_ プレフィックス
is_active    BOOLEAN  -- アクティブかどうか
is_deleted   BOOLEAN  -- 削除済みかどうか（論理削除）
has_children BOOLEAN  -- 子供がいるかどうか
can_comment  BOOLEAN  -- コメントできるかどうか
```

### 日時カラム

```sql
-- 作成・更新・削除は _at サフィックス
created_at  DATETIME  -- 作成日時
updated_at  DATETIME  -- 更新日時
deleted_at  DATETIME  -- 削除日時（論理削除）
published_at DATETIME -- 公開日時
expired_at  DATETIME  -- 有効期限

-- 日付のみは _date サフィックス（_on も使われる）
birth_date  DATE      -- 生年月日
event_date  DATE      -- イベント日

-- 期間・カウントは _count / _num / _amount
order_count INT       -- 注文数
total_amount DECIMAL  -- 合計金額
```

### ステータス・種別カラム

```sql
-- type, status, kind など（チーム内で統一する）
status        TINYINT   -- 状態（0:未確認, 1:処理中, 2:完了, etc.）
order_type    TINYINT   -- 注文種別
payment_method TINYINT  -- 支払い方法
```

### 数量・金額

```sql
quantity    INT UNSIGNED      -- 数量
price       DECIMAL(10,2)     -- 価格
tax_rate    DECIMAL(5,4)      -- 税率（例: 0.1000 = 10%）
total_amount DECIMAL(12,2)    -- 合計
```

---

## 5. インデックス名の命名規則

```sql
-- 単一インデックス：idx_{テーブル名}_{カラム名}
INDEX idx_orders_user_id (user_id)

-- 複合インデックス：idx_{テーブル名}_{カラム1}_{カラム2}
INDEX idx_orders_user_status (user_id, status)

-- UNIQUE制約：uq_{テーブル名}_{カラム名}
UNIQUE KEY uq_users_email (email)

-- 外部キー：fk_{テーブル名}_{参照先テーブル名}
CONSTRAINT fk_orders_users FOREIGN KEY (user_id) REFERENCES users(id)
```

---

## 6. 予約語・避けるべき名前

### SQLの予約語は使わない

```sql
-- ❌ SQL予約語
name, status, order, value, key, desc, group, user
-- MySQLではバッククォートで回避できるが混乱のもと

-- ✅ 回避方法
full_name, order_status, total_value, user_account
```

### 曖昧な名前は避ける

```sql
-- ❌ 曖昧
flag1, flag2          -- 何のフラグか不明
date1, date2          -- 何の日付か不明
num                   -- 何の数か不明
text                  -- 何のテキストか不明

-- ✅ 明確
is_premium, is_notified
start_date, end_date
retry_count
description
```

---

## 7. 実務でよく見る命名パターン集

```sql
-- ユーザー系
users
  id, email, password_digest, name, role, status,
  created_at, updated_at, deleted_at

-- 商品系
products
  id, name, description, price, stock_quantity,
  category_id, is_published, created_at, updated_at

-- 注文系
orders
  id, user_id, status, total_amount, note,
  ordered_at, shipped_at, delivered_at, cancelled_at,
  created_at, updated_at

-- 設定・メタ系（key-value）
settings
  id, user_id, key, value, created_at, updated_at

-- 通知系
notifications
  id, user_id, type, title, body, is_read,
  created_at, read_at
```

---

## 8. チームで命名規則を決める際のポイント

**最初に決めるべきこと：**

1. テーブル名：単数形 or 複数形（どちらかに統一）
2. 日時カラム：`_at` or `_on` or `_time`
3. ステータス：TINYINT + 定数 or ENUM
4. 論理削除：`deleted_at` or `is_deleted`
5. マスタテーブルのプレフィックス：あり or なし
6. 外部キー名：どのパターンで統一するか

**重要：プロジェクト開始前に決めてドキュメント化する。後から変えるのはコストが高い。**

---

## 9. まとめ

- テーブル・カラム名は `snake_case` の英語で統一する
- テーブル名は**複数形**、カラムの意味が名前から明確にわかるようにする
- `id`, `_id`, `is_`, `has_`, `_at`, `_date`, `_count`, `_amount` のサフィックス/プレフィックスを使いこなす
- 予約語・曖昧な名前（flag1, data, temp）は避ける
- チームでルールを決めてドキュメント化する

---

**次のステップ：[07_design_patterns.md](./07_design_patterns.md)** → 実務でよく出るケースと設計パターン
