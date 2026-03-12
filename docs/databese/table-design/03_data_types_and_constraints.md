# 03. データ型と制約：適切な型と整合性の守り方

---

## 0. まず結論（実務で迷ったらこれ）

DBは多様ですが、この章は **MySQL想定**（記法・挙動）でまとめます。

### よくある“基本セット”

- **ID**: `BIGINT UNSIGNED`（規模が小さければ `INT UNSIGNED` でも可）
- **金額**: `DECIMAL(10,2)`（または最小単位で `BIGINT` にする設計も有力）
- **真偽値/フラグ**: `TINYINT(1)`（MySQLの `BOOLEAN` は実体は `TINYINT(1)`）
- **文字列**: `VARCHAR(N)`（検索/ユニーク/インデックスが要るならまずこれ）
- **長文**: `TEXT`（検索に使わない本文・コメントなど）
- **日時**: `DATETIME`（タイムゾーンはアプリ側含め **UTC統一**が定番）

この「結論セット」から外れるときだけ、理由（要件・性能・運用）を言語化すると設計が安定します。

---

## 1. データ型の選び方が重要な理由

- **ストレージ効率**：`VARCHAR(255)` と `TEXT` は同じに見えて動作が違う
- **パフォーマンス**：数値型の比較は文字列型より速い
- **データ整合性**：型が正しければ不正な値が入りにくい（`INT` に文字は入らない）
- **将来の変更コスト**：後から型を変えると既存データの移行が大変

---

## 2. 数値型

### 整数型

| 型 | バイト | 範囲（符号あり） | 使いどころ |
|----|--------|-----------------|-----------|
| `TINYINT` | 1 | -128 〜 127 | フラグ、ステータス（0/1/2など） |
| `SMALLINT` | 2 | -32,768 〜 32,767 | 小さなマスタのID |
| `MEDIUMINT` | 3 | -8,388,608 〜 8,388,607 | あまり使わない |
| `INT` | 4 | -2,147,483,648 〜 2,147,483,647 | **最もよく使う汎用整数** |
| `BIGINT` | 8 | 約 -922京 〜 922京 | 超大規模テーブルのID、Unix timestamp |

**実務での選び方：**

- **原則**: 「意味に合う最小」よりも、IDは将来の拡張を見て **`INT` か `BIGINT` のどちらかに寄せる**（運用・可搬性が上がる）
- **`UNSIGNED`**: IDやカウントなど負にならないものは付ける（最大値が増える）

```sql
-- 一般的なIDカラム（21億件以内なら十分）
id INT UNSIGNED AUTO_INCREMENT

-- 大規模サービスや分散環境
id BIGINT UNSIGNED AUTO_INCREMENT

-- boolean的なフラグ（MySQLにはBOOLEANはあるが内部はTINYINT）
is_active TINYINT(1) NOT NULL DEFAULT 1
```

> 補足: `AUTO_INCREMENT` の採番方式や分散ID（ULID/UUID等）は別トピックなので、ここでは「型選び」に集中します。

### 小数点型

| 型 | 説明 | 使いどころ |
|----|------|-----------|
| `DECIMAL(M, D)` | 固定小数点。精度が保証される | **金額・税率・通貨** |
| `FLOAT` | 浮動小数点（精度低い） | 科学計算など誤差が許容できる場合 |
| `DOUBLE` | 浮動小数点（精度やや高い） | 同上 |

**金額には必ず `DECIMAL` を使う：**
```sql
price DECIMAL(10, 2)  -- 最大 99,999,999.99 円
-- FLOAT は計算誤差が出るため金額に使ってはいけない
```

#### もう1つの定番：最小単位で整数管理（オプション）

「小数が出ない通貨」や「端数処理ルールが明確」なら、**円やセントを整数で持つ**設計もよく使われます。

```sql
price_cents BIGINT UNSIGNED NOT NULL  -- 例: 1099 = $10.99
```

メリットは計算が簡単で誤差が出ないこと、デメリットは表示や通貨対応を設計に織り込む必要があることです。

---

## 3. 文字列型

| 型 | 説明 | 使いどころ |
|----|------|-----------|
| `CHAR(N)` | 固定長文字列（常にN文字埋められる） | 国コード(`JP`)・通貨コード(`JPY`)・固定長のコード類 |
| `VARCHAR(N)` | 可変長文字列（最大N文字） | **名前・タイトル・URLなど汎用** |
| `TEXT` | 最大65,535バイト | 本文・コメントなど長文 |
| `MEDIUMTEXT` | 最大16MB | ブログ記事・HTMLなど非常に長い文 |
| `LONGTEXT` | 最大4GB | ほぼ使わない |
| `JSON` | JSON形式（MySQL 5.7+） | スキーマが不定なデータ |

**VARCHAR の長さの考え方：**
```sql
name VARCHAR(100)       -- 人名（日本語氏名は普通10文字以内）
email VARCHAR(255)      -- メールアドレスの最大仕様が254文字
url VARCHAR(2083)       -- URLの最大長
title VARCHAR(200)      -- 記事タイトルなど
description VARCHAR(500) -- 短い説明文
```

**TEXT か VARCHAR か：**
- インデックスを貼りたい → `VARCHAR`（`TEXT` も prefix index 等で対応できるが制約が増える）
- WHERE句で使わない長文 → `TEXT`
- 255文字以内 → `VARCHAR`

### よくある落とし穴

- **電話番号・郵便番号は数値にしない**（先頭0、`+81`、ハイフン、将来の形式変更がある）
  - 例: `phone_number VARCHAR(20)`
- **「文字数」ではなく「バイト数」制約に注意**
  - 文字コード（例: `utf8mb4`）だと1文字が複数バイトになり得る
  - インデックス長の上限（MySQL設定/バージョン依存）にも影響する

---

## 4. 日時型

| 型 | 説明 | 範囲 | 使いどころ |
|----|------|------|-----------|
| `DATE` | 日付のみ | 1000-01-01 〜 9999-12-31 | 生年月日、イベント日 |
| `DATETIME` | 日付+時刻 | 1000-01-01 〜 9999-12-31 | **作成日時・更新日時など汎用** |
| `TIMESTAMP` | UNIXタイムスタンプ | 1970-01-01 〜 2038-01-19 | **自動更新に便利だが2038年問題あり** |
| `TIME` | 時刻のみ | -838:59:59 〜 838:59:59 | 営業時間など |
| `YEAR` | 年のみ | 1901 〜 2155 | 会計年度など |

**実務での使い方：**
```sql
-- 推奨：作成・更新日時は DATETIME で管理
created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP

-- TIMESTAMP は2038年問題があるため新規設計では DATETIME を推奨
```

> 注意: `DATETIME` の `DEFAULT CURRENT_TIMESTAMP` は MySQL 5.6.5 以降で利用可能です。古い環境では `TIMESTAMP` しか使えない等の制約があるので、DBバージョンを前提に判断します。

**タイムゾーン問題：**
- `TIMESTAMP` はDBサーバーのタイムゾーンに依存する
- `DATETIME` はタイムゾーン非依存（入れた値がそのまま入る）
- グローバルサービスでは UTC で統一するのが一般的

---

## 5. その他の型

```sql
-- 真偽値（MySQLはBOOLEANはTINYINT(1)のエイリアス）
is_deleted BOOLEAN NOT NULL DEFAULT FALSE  -- または TINYINT(1)

-- 列挙型（固定の選択肢）
status ENUM('active', 'inactive', 'pending') NOT NULL DEFAULT 'pending'
-- ※ ENUMはカラム変更が重いため、TINYINT + 定数定義の方が好まれることも多い

-- バイナリ（画像・バイナリデータ）
-- DBに入れるのは基本的に非推奨（S3などに置いてURLをVARCHARで持つ）
```

---

## 6. 制約（Constraints）の種類

制約は「不正なデータが入らないようにする仕組み」。アプリ側でのバリデーションと二重で守ることで堅牢なシステムになる。

### PRIMARY KEY（主キー制約）

```sql
id INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY
```

- レコードを一意に識別する
- NULL不可 + UNIQUE が自動的に設定される
- テーブルに1つだけ設定できる

### NOT NULL制約

```sql
name VARCHAR(100) NOT NULL
```

- NULL を禁止する
- **迷ったら NOT NULL（ただし意味がある場合はNULLも使う）**
- NULL は「値が不明/存在しない」を表すため、要件が曖昧なまま許容するとバグの温床になりやすい

**NULL を許容する場合の判断基準：**
- 「まだ値が存在しない」ことを表現したい場合のみ許容
- 例：`deleted_at`（削除されていなければNULL）、`approved_at`（未承認はNULL）

### UNIQUE制約

```sql
email VARCHAR(255) NOT NULL UNIQUE
```

- カラム（または複数カラムの組み合わせ）で重複を禁止する
- 自動的にインデックスが作成される

```sql
-- 複合UNIQUEの例（同じユーザーが同じ商品を複数回フォローしないように）
UNIQUE KEY uq_user_product (user_id, product_id)
```

### DEFAULT制約

```sql
status TINYINT NOT NULL DEFAULT 1,
created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
```

- 値を指定しなかった場合のデフォルト値を設定する

### CHECK制約（MySQL 8.0.16+）

```sql
age INT NOT NULL CHECK (age >= 0 AND age <= 150),
price DECIMAL(10,2) NOT NULL CHECK (price >= 0)
```

- 指定した条件を満たすデータのみ許可する

### FOREIGN KEY（外部キー制約）

```sql
FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE ON UPDATE CASCADE
```

- 参照先のテーブルに存在しない値を入れることを禁止する
- 詳細は [04_keys_and_relationships.md](./04_keys_and_relationships.md) で解説

---

## 7. NULL との正しい付き合い方

```sql
-- NULLの比較は = ではなく IS NULL / IS NOT NULL を使う
SELECT * FROM users WHERE deleted_at IS NULL;

-- COUNT(*)とCOUNT(カラム名)の違い
COUNT(*)         -- NULL含む全行数
COUNT(email)     -- emailがNULLでない行数
```

**NULL が設計に与える影響：**

| 問題 | 説明 |
|------|------|
| 比較の難しさ | `NULL = NULL` は常にFALSE → `IS NULL`が必要 |
| 集計のずれ | `SUM()`, `AVG()` はNULLを無視する |
| JOINの問題 | 外部結合でNULLが発生しやすく意図しない結果に |
| インデックス効率低下 | NULLが多いとインデックスの効率が下がることがある |

---

## 8. カラム設計のチェックリスト

```
□ 数値は文字列型で保存していないか（電話番号除く）
□ 金額はDECIMALを使っているか（FLOATではないか）
□ 日時はDATETIMEを使っているか（2038年問題回避）
□ NOT NULLを適切につけているか
□ UNIQUEが必要なカラムに制約が付いているか
□ VARCHARの長さは適切か（大きすぎ・小さすぎはないか）
□ ENUMを使う場合、将来の拡張に対応できるか
□ NULLを許容する場合、その意味が明確か
```

---

## 9. まとめ

- データ型は「意味に合った最小の型」を選ぶ
- 金額は `DECIMAL`、日時は `DATETIME`、IDは `INT/BIGINT UNSIGNED`
- NOT NULL を基本とし、NULL を使う場合は意図を明確にする
- 制約はアプリ側バリデーションと二重に設定することでデータの整合性を守る

---

**次のステップ：[04_keys_and_relationships.md](./04_keys_and_relationships.md)** → テーブル間のつながりと主キー・外部キーの設計
