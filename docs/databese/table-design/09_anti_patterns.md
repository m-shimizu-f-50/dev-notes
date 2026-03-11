# 09. アンチパターン：やってはいけない設計とその理由

---

## 1. ジェイウォーク（Jaywalking）：カンマ区切りで複数値を1セルに詰める

### NG例

```sql
-- ❌ 商品のタグをカンマ区切りで1カラムに入れる
products
| id | name    | tags                    |
|----|---------|-------------------------|
|  1 | りんご  | 果物,国産,有機          |
|  2 | バナナ  | 果物,輸入品             |
```

### 問題点

```sql
-- 「果物」タグを持つ商品を検索する → 非常に困難
SELECT * FROM products WHERE tags LIKE '%果物%';
-- LIKE検索はインデックスが使えない（前方一致以外）

-- タグ数のカウントが難しい
-- 特定のタグを削除するのが困難
-- タグが増えたとき文字列長の上限問題が起きる
```

### 正しい設計

```sql
-- 中間テーブルで多対多
product_tags
| product_id | tag_id |
```

---

## 2. フィアオブジアンノウン（Fear of the Unknown）：NULLを適切に使わない

### NG例①：NULLの代わりに意味のない値を入れる

```sql
-- ❌ 「不明」を空文字・0・'N/A'で代替する
phone VARCHAR(20) NOT NULL DEFAULT ''     -- 電話番号がない場合に空文字
age   INT NOT NULL DEFAULT -1            -- 年齢不明を -1 で表現
```

**問題：** `-1` の年齢や空文字の電話番号は「不明」なのかそういう値なのか区別できない

### NG例②：NULLを恐れてすべてNOT NULLにする

```sql
-- ❌ 論理削除の deleted_at が必ず値を必要とする設計
deleted_at DATETIME NOT NULL DEFAULT '1970-01-01 00:00:00'
-- → 「削除されていない」を表すダミー日時が入っている（意味的に不正確）

-- ✅ NULL を正しく使う
deleted_at DATETIME NULL DEFAULT NULL  -- NULLは「削除されていない」を意味する
```

### 正しいアプローチ

- 「値がない」ことを表現したいときは NULL を使う
- ただし NULL の多用はクエリを複雑にするため、本当に必要な場合だけ使う

---

## 3. 魔法の値（Magic Beans）：意味不明な数値を使う

### NG例

```sql
-- ❌ status の 0, 1, 2, 3 が何を意味するか不明
status TINYINT NOT NULL DEFAULT 0
```

テーブル定義だけ見ても、0が「有効」なのか「無効」なのか「未処理」なのかわからない。

### 正しい設計

```sql
-- ✅ コメントで意味を記述する
status TINYINT NOT NULL DEFAULT 0
  COMMENT '0:下書き 1:公開中 2:非公開 3:削除済み'

-- またはアプリ側で定数化する
-- ORDER_STATUS_DRAFT     = 0
-- ORDER_STATUS_PUBLISHED = 1
-- ORDER_STATUS_PRIVATE   = 2
-- ORDER_STATUS_DELETED   = 3
```

---

## 4. なんでもVARCHAR：型を無視した設計

### NG例

```sql
-- ❌ すべての値を VARCHAR で保存する
price       VARCHAR(20)  -- 数値なのに文字列
is_active   VARCHAR(5)   -- boolean なのに文字列
created_at  VARCHAR(30)  -- 日時なのに文字列
user_id     VARCHAR(10)  -- 整数IDなのに文字列
```

### 問題点

```sql
-- 金額の大小比較が正しく動かない（文字列の辞書順）
ORDER BY price ASC
-- '100', '200', '9' → '100', '200', '9' の順（9 > 200 になってしまう）

-- 日時の範囲検索が正しく動かない（フォーマット依存）
WHERE created_at > '2024-01-01'
-- 文字列比較になるためインデックスも機能しない

-- 不正な値が入る
INSERT INTO products (price) VALUES ('安い');  -- 通ってしまう
```

---

## 5. 巨大な単一テーブル（God Table）

### NG例

```sql
-- ❌ あらゆる情報を1テーブルに詰め込む
users
| id | name | email | bio | avatar | facebook_id | twitter_id |
| stripe_customer_id | last_login | login_count | role | is_admin |
| address | city | country | zip | phone | ...（50カラム超）
```

### 問題点

- 読み取りのたびに使わないカラムも取得される
- テーブルが肥大化しメモリ効率が悪い
- 変更が怖くなる（影響範囲が大きい）

### 正しい設計

```sql
-- 役割ごとに分割する
users             -- 認証情報（email, password_digest）
user_profiles     -- プロフィール（bio, avatar）
user_addresses    -- 住所
user_social_links -- SNSアカウント
user_settings     -- 設定情報
```

---

## 6. 連番カラム（Repeating Columns）

### NG例

```sql
-- ❌ 将来の拡張のために列を複数用意しておく
products
| id | name | image1 | image2 | image3 | image4 | image5 |
```

### 問題点

- 「5枚まで」という制約がDB設計で表現されてしまっている
- 将来6枚目が必要になったときにカラム追加が必要
- 画像が1枚しかないときも image2〜5 がNULLで無駄に領域確保

### 正しい設計

```sql
-- 別テーブルで1対多にする
CREATE TABLE product_images (
  id          INT UNSIGNED NOT NULL AUTO_INCREMENT,
  product_id  INT UNSIGNED NOT NULL,
  image_url   VARCHAR(500) NOT NULL,
  sort_order  INT UNSIGNED NOT NULL DEFAULT 0,
  PRIMARY KEY (id),
  INDEX idx_product_id (product_id)
);
```

---

## 7. EAV（Entity-Attribute-Value）の乱用

### EAVとは

```sql
-- 属性名と値をKV形式で汎用的に保存するパターン
CREATE TABLE product_attributes (
  product_id      INT UNSIGNED NOT NULL,
  attribute_name  VARCHAR(100) NOT NULL,
  attribute_value VARCHAR(500)
);

-- データ例
-- (1, 'color', 'red')
-- (1, 'weight', '500g')
-- (1, 'size', 'M')
```

### 問題点

```sql
-- ❌ 特定の属性での検索・ソートが大変
SELECT * FROM products
JOIN product_attributes pa1 ON products.id = pa1.product_id AND pa1.attribute_name = 'color'
JOIN product_attributes pa2 ON products.id = pa2.product_id AND pa2.attribute_name = 'size'
WHERE pa1.attribute_value = 'red' AND pa2.attribute_value = 'M';
-- → JOINが増えて複雑かつ遅い

-- データ型の保証ができない（すべてVARCHAR）
-- 必須属性の強制ができない（NOT NULL が機能しない）
```

### EAVを使ってもよいケース

- 本当に属性のスキーマが動的で予測不可能な場合
- その場合は MySQL 5.7+/PostgreSQL の `JSON` カラムを検討する方が現代的

```sql
-- JSONカラムによる柔軟なメタデータ（EAVより現代的）
attributes JSON NULL COMMENT '商品固有の属性（{"color": "red", "size": "M"}）'
```

---

## 8. 外部キーなしで整合性を無視する

### NG例

```sql
-- ❌ 外部キー制約を省略したまま運用
CREATE TABLE orders (
  user_id INT UNSIGNED NOT NULL
  -- FOREIGN KEY を設定していない
);

-- → 存在しないuser_idのorderが作れてしまう
-- → usersを削除してもordersが残る（孤立レコード）
```

### 問題点

- 孤立レコードが蓄積していく
- バグの発見が遅れる（アプリが壊れるまで気づかない）

### 正しい設計

```sql
FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE RESTRICT
```

> ただし大規模・分散システムでFK制約を使わないケースについては `04_keys_and_relationships.md` を参照。

---

## 9. データをファイルパスで管理しない（画像URLをDB外で管理）

### NG例

```sql
-- ❌ ローカルファイルパスをDB保存する
avatar_path VARCHAR(500)  -- '/var/www/uploads/user_123_avatar.jpg'
```

### 問題点

- サーバー移転・スケールアウト時に壊れる
- ファイルとDBの整合性を保つのが困難

### 正しい設計

```sql
-- ✅ S3などのオブジェクトストレージのURLかキーを保存
avatar_url VARCHAR(500)   -- 'https://s3.ap-northeast-1.amazonaws.com/bucket/user_123.jpg'
-- または
avatar_key VARCHAR(200)   -- 'users/123/avatar.jpg' （ベースURLはアプリ側で補完）
```

---

## 10. アンチパターン早見表

| アンチパターン | 問題 | 解決策 |
|--------------|------|--------|
| カンマ区切り | 検索不可・インデックス無効 | 中間テーブルで多対多 |
| なんでもVARCHAR | 型の保証なし・ソート不正 | 適切なデータ型を使う |
| 巨大テーブル | 変更困難・非効率 | 役割ごとにテーブルを分割 |
| 連番カラム | 上限固定・拡張困難 | 1対多テーブルに分離 |
| EAV乱用 | 検索困難・型保証なし | 固定スキーマかJSONカラム |
| 外部キーなし | 孤立レコード・整合性崩壊 | FK制約を設定する |
| ローカルファイルパス | スケール困難 | S3等のURLを保存 |
| 魔法の値 | 可読性ゼロ | コメントで意味を記述 |

---

## 11. まとめ

アンチパターンを避けるための3つの問い：

1. **このカラムに入る値のパターンは？** → 型と制約を適切に決める
2. **このデータでどんな検索・集計を行うか？** → インデックスと正規化を考える
3. **1年後にデータが10倍になったときどうなるか？** → 拡張性を考える

---

**おつかれさまでした！**
全9章を読んだあなたは、実務で通用するテーブル設計の基礎を身につけています。
次は実際にアプリを作りながら設計してみて、レビューを受けることで実力が定着します。
