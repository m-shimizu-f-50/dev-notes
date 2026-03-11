# 05. インデックス設計：検索を速くする仕組み

---

## 1. インデックスとは

インデックスとは「本の索引」のようなもの。本文全体を読まなくても、目的のページをすぐ見つけられるようにする仕組み。

```
インデックスなし（フルスキャン）：
全レコードを1件ずつ順番に探す → 100万件なら100万回チェック

インデックスあり（インデックススキャン）：
B-Treeで絞り込み → 100万件でも対数時間（log2(100万) ≒ 20回）
```

---

## 2. インデックスが自動的に作られるもの

以下の制約・設定をつけると、自動でインデックスが作成される：

```sql
-- PRIMARY KEY → 自動的にインデックスが作られる
id INT UNSIGNED PRIMARY KEY

-- UNIQUE → 自動的にインデックスが作られる
email VARCHAR(255) UNIQUE

-- FOREIGN KEY → MySQLでは自動でインデックスが作られる（PostgreSQLは手動）
FOREIGN KEY (user_id) REFERENCES users(id)
```

---

## 3. 手動でインデックスを貼る

```sql
-- 単一インデックス
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- 複合インデックス
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- テーブル作成時に一緒に定義する方法
CREATE TABLE orders (
  id      INT UNSIGNED NOT NULL AUTO_INCREMENT,
  user_id INT UNSIGNED NOT NULL,
  status  TINYINT NOT NULL DEFAULT 0,
  PRIMARY KEY (id),
  INDEX idx_user_id (user_id),
  INDEX idx_user_status (user_id, status)
);
```

---

## 4. インデックスを貼るべき場所

### ルール：「WHERE句・ORDER BY・JOINで使うカラム」に貼る

```sql
-- よく使うWHERE条件
SELECT * FROM orders WHERE user_id = 100;
-- → user_id にインデックスが必要

-- よく使うソート
SELECT * FROM articles ORDER BY published_at DESC;
-- → published_at にインデックスが必要

-- JOINの結合条件
SELECT * FROM orders JOIN users ON orders.user_id = users.id;
-- → orders.user_id にインデックスが必要（usersのidはPKなので自動でOK）
```

### 実務での判断フロー

```
1. そのクエリは頻繁に実行されるか？
   ↓ Yes
2. テーブルに十分なデータがあるか（目安：1000件以上）？
   ↓ Yes
3. EXPLAIN で実行計画を確認する
   ↓ フルスキャンになっている
4. インデックスを追加する
```

---

## 5. 複合インデックスの重要ルール

### 左端一致の原則（Leftmost Prefix Rule）

```sql
-- (user_id, status, created_at) の複合インデックスがある場合

-- ✅ インデックスが使われる
WHERE user_id = 100
WHERE user_id = 100 AND status = 1
WHERE user_id = 100 AND status = 1 AND created_at > '2024-01-01'

-- ❌ インデックスが使われない（user_idを飛ばしている）
WHERE status = 1
WHERE created_at > '2024-01-01'
WHERE status = 1 AND created_at > '2024-01-01'
```

**複合インデックスの列順の決め方：**
1. **選択性の高いカラム（値の種類が多いカラム）** を左に置く
2. **等値条件（=）で使うカラム** を範囲条件（>、<、BETWEEN）より左に置く

```sql
-- 例：status（0,1,2の3種類）よりuser_id（多数）の方が選択性が高い
INDEX idx_user_status (user_id, status)   -- ✅ 良い順番
INDEX idx_status_user (status, user_id)   -- 微妙な順番
```

---

## 6. EXPLAIN で実行計画を確認する

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 100;
```

**重要なカラムの見方：**

| カラム | 良い状態 | 悪い状態 |
|--------|---------|---------|
| `type` | `ref`, `range`, `eq_ref`, `const` | `ALL`（フルスキャン） |
| `key` | インデックス名が表示される | `NULL`（インデックス未使用） |
| `rows` | 少ない数 | 全レコード数に近い数 |
| `Extra` | `Using index`（カバリングインデックス） | `Using filesort`, `Using temporary` |

```sql
-- 例：typeが ALL になっていたらインデックスが使われていない
id | select_type | table  | type | key  | rows    | Extra
---+-------------+--------+------+------+---------+-------
 1 | SIMPLE      | orders | ALL  | NULL | 1000000 | Using where
```

---

## 7. インデックスが使われないケース（落とし穴）

### ❌ カラムに関数を適用する

```sql
-- NG：インデックスが使われない
WHERE YEAR(created_at) = 2024
WHERE LOWER(email) = 'test@example.com'

-- OK：インデックスが使われる
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
WHERE email = 'test@example.com'  -- 入力を小文字に統一しておく
```

### ❌ 型が一致していない

```sql
-- user_id が INT なのに文字列で比較
WHERE user_id = '100'  -- NG（暗黙の型変換が起きてインデックス無効）
WHERE user_id = 100    -- OK
```

### ❌ LIKE の前方一致以外

```sql
-- NG：後方・中間一致はインデックスが使えない
WHERE name LIKE '%田中%'
WHERE name LIKE '%田中'

-- OK：前方一致はインデックスが使える
WHERE name LIKE '田中%'
```

### ❌ OR 条件で一部のカラムにインデックスがない

```sql
-- user_id にはインデックスがあるが email にはない場合
WHERE user_id = 100 OR email = 'test@example.com'
-- → どちらかがフルスキャンになるため全体がフルスキャンになる
```

### ❌ NULL 判定

```sql
-- NULL の = 比較はインデックスが使われないことがある
WHERE deleted_at IS NULL  -- DBによる（MySQLは使える場合もある）
-- → deleted_at に 0 や '1970-01-01' を使う設計も検討する
```

---

## 8. カバリングインデックス

クエリで必要なカラムがすべてインデックスに含まれる場合、テーブル本体へのアクセスが不要になり高速になる。

```sql
-- EXPLAIN の Extra に "Using index" と表示されれば成功

-- orders(user_id, status, created_at) という複合インデックスがある場合
SELECT user_id, status, created_at FROM orders WHERE user_id = 100;
-- → テーブル本体を見ずにインデックスだけで完結（高速）
```

---

## 9. インデックスの副作用（貼りすぎに注意）

```
インデックスのデメリット：
1. 書き込み（INSERT/UPDATE/DELETE）が遅くなる
   → インデックスの更新も同時に行うため

2. ディスク容量を消費する
   → 大きなテーブルにインデックスを大量に貼ると数GBになることも

3. クエリオプティマイザが混乱することがある
   → 適切でないインデックスがあると逆に遅くなることも
```

**目安：読み取りが多いシステムはインデックスを積極的に、書き込みが多いシステムは最小限に。**

---

## 10. インデックス設計のチェックリスト

```
□ WHERE句でよく使うカラムにインデックスがあるか
□ JOINの結合カラム（FK側）にインデックスがあるか
□ ORDER BY や GROUP BY で使うカラムにインデックスがあるか
□ 複合インデックスの順序は適切か（左端一致の原則）
□ EXPLAIN でフルスキャン(ALL)が発生していないか
□ インデックスを貼りすぎていないか（書き込みパフォーマンスとのトレードオフ）
□ 関数をWHERE句で使って無効化していないか
```

---

## 11. まとめ

- インデックスは「WHERE / ORDER BY / JOIN」のカラムに貼る
- 複合インデックスは左端一致の原則に従って列順を決める
- `EXPLAIN` で実行計画を確認し、`type=ALL` はアンチパターン
- カラムへの関数適用・型不一致・LIKE前方一致以外はインデックスが使われない
- インデックスは貼りすぎると書き込みパフォーマンスが落ちる

---

**次のステップ：[06_naming_conventions.md](./06_naming_conventions.md)** → チームで読めるテーブル・カラム名の付け方
