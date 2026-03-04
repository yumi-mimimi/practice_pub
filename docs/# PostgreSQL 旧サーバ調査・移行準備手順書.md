# PostgreSQL 旧サーバ調査・移行準備手順書 

## 1. この手順書の目的

この手順書は、旧サーバー上の PostgreSQL を**変更せずに調査**し、リプレイス/移行の準備に必要な情報を安全に集めるためのものです。

この手順書で実施することは、主に次の4つです。

1. 旧DBの構成を把握する
2. 移行先で再現すべき設定を特定する
3. 容量の主因がテーブル本体・インデックス・TOAST のどれかを切り分ける
4. テスト用にテーブル定義やダミーデータを再現できる状態にする

---

## 2. この手順書の前提

- 旧サーバーへ SSH 接続できること
- 旧サーバー上の PostgreSQL に `psql` で接続できること
- 調査対象は旧サーバー側 PostgreSQL であること
- テストデータ作成や DDL 再現は**新サーバーまたは別の検証環境**で行うこと
- 旧サーバーでは**変更系操作を行わない**こと

---

## 3. プレースホルダの書き方ルール

この手順書では、値を差し替える箇所を次のように表記します。

- `<調査対象DB名>`
  - 例: `dash_replace`
  - 意味: 旧サーバーで調査したい DB 名
- `<調査対象スキーマ名>`
  - 例: `public`
  - 意味: テーブルが所属するスキーマ名
- `<調査対象テーブル名>`
  - 例: `articles`
  - 意味: 調査したい実テーブル名
- `<対象テーブルハッシュ>`
  - 例: `f698d22a22`
  - 意味: 匿名化して使う 10 文字ハッシュ
- `<テスト用スキーマ名>`
  - 例: `test_schema`
  - 意味: 新サーバーや検証環境で使うスキーマ名
- `<テスト用テーブル名>`
  - 例: `toast_test`
  - 意味: 新サーバーや検証環境で作るテーブル名
- `<投入行数>`
  - 例: `100000`
  - 意味: テストで投入したい件数
- `<1行あたりの文字数>`
  - 例: `3501`
  - 意味: 1 行ごとの text サイズを何文字にするか

### 重要な補足

- `CREATE TABLE` に指定するのは **DB 名ではなくテーブル名** です。
- `schema.table` の形式で書くと、どのスキーマに作るか明確になります。
- SQL の識別子として使う値は、通常は**シングルクォートで囲みません**。
  - 例: `CREATE TABLE <テスト用スキーマ名>.<テスト用テーブル名> (...)`
- ただし、`pg_dump -t 'schema.table'` のように**シェル引数として渡す箇所**ではシングルクォートを使うことがあります。

---

## 4. 安全方針

旧サーバー上では、次を徹底します。

- 調査は読み取り専用セッションで行う
- 個人情報や固有名は外部共有時に伏せる
- 変更系コマンドは実行しない

実行しない代表例:

- `INSERT`
- `UPDATE`
- `DELETE`
- `ALTER`
- `DROP`
- `TRUNCATE`
- `VACUUM FULL`
- `REINDEX`
- `CREATE TABLE`（旧サーバー上では行わない）

### 読み取り専用設定に関する注意

`SET default_transaction_read_only = on;` は**その接続セッションだけ**に効きます。

そのため、`\c` で別 DB に接続し直した場合は、**再度読み取り専用を設定**してください。

---

## 5. 全体の流れ

1. 旧サーバーに入り `psql` を起動する
2. 調査用セッションを読み取り専用にする
3. DB 一覧を匿名化して確認する
4. 本命 DB に接続して再度読み取り専用にする
5. バージョン・文字コード・照合順序・拡張を確認する
6. スキーマとテーブルの構成・容量を匿名化して確認する
7. 最大テーブルの容量内訳を確認する
8. TOAST が膨張なのか実データなのかを確認する
9. 巨大データを持つ列の候補を調べる
10. 元テーブル定義の確認と DDL 取得を行う
11. 新サーバー/検証環境でテストデータを作る

---

## 6. 旧サーバーへ入り、psql を起動する

### 目的

- PostgreSQL の管理ユーザーとしてログインし、調査を始める
- どの DB に入っているかを確認する

### 実行コマンド

```bash
sudo -iu postgres
psql
```

接続確認:

```sql
\conninfo
```

### 期待される出力例

```text
You are connected to database "postgres" as user "postgres" via socket in "/tmp" at port "5432".
```

### 出力結果の意味

- `database "postgres"`
  - 今どの DB に接続しているか
- `user "postgres"`
  - どの PostgreSQL ロールで接続しているか
- `via socket in "/tmp"`
  - ローカルソケット接続であること
- `port "5432"`
  - 接続先ポート番号

### 次にやること

- このあと読み取り専用設定を有効にする

---

## 7. 調査セッションを読み取り専用にする

### 目的

- SQLを誤って実行/編集してしまう事故予防
- 調査対象の旧サーバーに影響を防ぐこと

### 実行コマンド

```sql
SHOW default_transaction_read_only;
SET default_transaction_read_only = on;
SHOW default_transaction_read_only;
```

### 期待される出力例

実行前:

```text
 default_transaction_read_only
-------------------------------
 off
(1 row)
```

設定後:

```text
 default_transaction_read_only
-------------------------------
 on
(1 row)
```

### 出力結果の意味

- `off`
  - 現在のセッションでは書き込み可能
- `on`
  - 現在のセッションではデフォルトで読み取り専用

### 判断ポイント

- 最後の結果が `on` になっていれば OK
- `\c` で別 DB に接続し直したら、もう一度同じ設定を入れる

---

## 8. DB 一覧を匿名化して確認する

### 目的

- サーバー上にあるDBの状況把握
- DBサイズ調査
- DB 名をそのまま外部共有予防

### 実行コマンド

```sql
SELECT
  row_number() OVER (ORDER BY pg_database_size(datname) DESC) AS id,
  substr(md5(datname), 1, 10) AS db_name_hash,
  pg_size_pretty(pg_database_size(datname)) AS size,
  datistemplate AS is_template,
  datallowconn AS allow_conn
FROM pg_database
WHERE datname NOT IN ('template0','template1')
ORDER BY pg_database_size(datname) DESC;
```

### 期待される出力例

```text
 id | db_name_hash |  size   | is_template | allow_conn
----+--------------+---------+-------------+------------
  1 | bc659ddfde   | 17 GB   | f           | t
  2 | e8a4865385   | 7715 kB | f           | t
(2 rows)
```

### 出力結果の意味

- `id`
  - サイズ順の通し番号
- `db_name_hash`
  - DB 名を匿名化した 10 文字ハッシュ※匿名のため本当の名前じゃないです。
- `size`
  - DB 全体のサイズ
- `is_template = f`
  - テンプレート DB ではない
- `allow_conn = t`
  - 接続可能な DB

### 今回の演習での読み方

- 一番大きい `17 GB` の DB がメインと推定
- 小さい DB は保守用・管理用・空に近い DB の可能性あり

### 次にやること

- ハッシュと実名の対応を、**自分の手元だけで**確認する

---

## 9. 本命 DB の実名を手元だけで確認する

### 目的

- 匿名化された本命 DB が、実際にはどの DB 名なのか自分だけで特定する
- 後続の調査対象 DB を決める

### 実行コマンド

```sql
SELECT datname, substr(md5(datname), 1, 10) AS db_name_hash
FROM pg_database
WHERE datname NOT IN ('template0','template1')
ORDER BY pg_database_size(datname) DESC;
```

### 期待される出力例

```text
    datname    | db_name_hash
---------------+--------------
 dash_replace  | bc659ddfde
 postgres      | e8a4865385
(2 rows)
```

### 出力結果の意味

- `datname`
  - DB の実名
- `db_name_hash`
  - 匿名化結果

### 注意

- この出力は外部共有はNGです。外部サイトに展開は危険です
- 実名を資料へ貼る必要がある場合は、社内ルールに従うこと

### 次にやること

- 本命 DB に接続する

---

## 10. 本命 DB に接続し、再度読み取り専用にする

### 目的

- 調査対象 DB に入る。項番で実施したコマンドはSQL全体情報を確認するためのものです。今回はテーブルに潜入して実施されます。そのためコマンドが重複してます。

- 切り替え後も安全のため読み取り専用にしておく

### 実行コマンド

```sql
\c <調査対象DB名>
\conninfo
SHOW default_transaction_read_only;
SET default_transaction_read_only = on;
SHOW default_transaction_read_only;
```

### 変数の意味

- `<調査対象DB名>`
  - 調査したい本命 DB 名

### 期待される出力例

```text
You are now connected to database "dash_replace" as user "postgres".

You are connected to database "dash_replace" as user "postgres" via socket in "/tmp" at port "5432".

 default_transaction_read_only
-------------------------------
 off
(1 row)

SET

 default_transaction_read_only
-------------------------------
 on
(1 row)
```

### 出力結果の意味

- `You are now connected ...`
  - DB の切り替えが完了した
- 最後の `on`
  - 新しい接続先でも読み取り専用が有効になった

### 次にやること

- その DB のバージョン、文字コード、照合順序を調べる

---

## 11. PostgreSQL バージョン・文字コード・照合順序を確認する

### 目的

- 移行先で再現すべき DB の基本設定を把握する
- 文字コードや locale 差異による不具合を防ぐ

### 実行コマンド

```sql
SELECT version();

SELECT
  current_database() AS db,
  pg_encoding_to_char(encoding) AS encoding,
  datcollate AS collate,
  datctype AS ctype
FROM pg_database
WHERE datname = current_database();
```

### 期待される出力例

`SELECT version();`

```text
                                                version
--------------------------------------------------------------------------------------------------------
 PostgreSQL 11.2 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 7.3.1 20180712 (Red Hat 7.3.1-6), 64-bit
(1 row)
```

`SELECT ... encoding / collate / ctype ...`

```text
      db      | encoding |   collate   |    ctype
--------------+----------+-------------+-------------
 dash_replace | UTF8     | en_US.UTF-8 | en_US.UTF-8
(1 row)
```

### 出力結果の意味

- `PostgreSQL 11.2`
  - 現在のメジャー/マイナーバージョン
- `encoding = UTF8`
  - DB の文字コード
- `collate = en_US.UTF-8`
  - 文字列の並び順や比較順序に関わる設定
- `ctype = en_US.UTF-8`
  - 文字種判定に関わる設定

### この演習では・・・

- 旧 DB は `UTF8` / `en_US.UTF-8`
- 移行先で locale が変わると、並び順やインデックスの挙動に影響する可能性がある
- PostgreSQL 11 は EOL のため、新サーバーでは新しめのメジャーバージョンを検討する必要があります

### 次にやること

- 拡張機能の有無を確認する

---

## 12. 拡張機能を確認する

### 目的

- 移行先で事前インストールが必要な拡張機能があるか確認する
- `pg_restore` 前に追加導入が必要か判断する

### 実行コマンド

```sql
SELECT extname, extversion, nspname AS schema
FROM pg_extension e
JOIN pg_namespace n ON n.oid = e.extnamespace
ORDER BY extname;
```

### 期待される出力例

```text
 extname | extversion | schema
---------+------------+--------
 plpgsql | 1.0        | public
(1 row)
```

### 出力結果の意味

- `extname`
  - 拡張機能名※変数です。取り扱っているサーバによって項目が変化します。
- `extversion`
  - 拡張バージョン
- `schema`
  - 拡張オブジェクトの配置先スキーマ

### この演習では

- `plpgsql` のみであれば、追加で特別な拡張を必要としていない可能性が高いです
- `postgis`, `pgcrypto`, `uuid-ossp`, `hstore`, `citext` などがある場合は、新サーバー側でも事前準備が必要

### 次にやること

- スキーマ構成を匿名化して確認する

---

## 13. スキーマ構成を匿名化して確認する

### 目的

- 非システムスキーマがいくつあるか確認する
- テーブル数、ビュー数、シーケンス数を把握する
- どのスキーマに業務データがいるか把握する

### 実行コマンド

```sql
WITH s AS (
  SELECT oid, nspname
  FROM pg_namespace
  WHERE nspname NOT LIKE 'pg_%'
    AND nspname <> 'information_schema'
)
SELECT
  row_number() OVER (ORDER BY nspname) AS id,
  substr(md5(nspname), 1, 10) AS schema_hash,
  (SELECT count(*) FROM pg_class c WHERE c.relnamespace = s.oid AND c.relkind = 'r') AS tables,
  (SELECT count(*) FROM pg_class c WHERE c.relnamespace = s.oid AND c.relkind IN ('v','m')) AS views,
  (SELECT count(*) FROM pg_class c WHERE c.relnamespace = s.oid AND c.relkind = 'S') AS sequences
FROM s
ORDER BY tables DESC, views DESC, sequences DESC, schema_hash;
```

### 期待される出力例

```text
 id | schema_hash | tables | views | sequences
----+-------------+--------+-------+-----------
  1 | a1b2c3d4e5  |     42 |     0 |        38
(1 row)
```

### 出力結果の意味

- `schema_hash`
  - スキーマ名の匿名化結果
- `tables`
  - 通常テーブル数
- `views`
  - ビューとマテリアライズドビュー数
- `sequences`
  - シーケンス数

### この研修での読み方

- 非システムスキーマが 1 つなので、業務データは 1 スキーマにまとまっている可能性が高い
- `tables = 42` は、テーブル調査のボリュームにあたリます

### 次にやること

- 容量の大きいテーブルから優先的に確認する

---

## 14. 容量の大きいテーブルを匿名化して確認する

### 目的

- 容量の大きいテーブルを把握する
- どのテーブルが移行時間に最も影響するかの計算に利用
- テーブル本体とそれ以外の比率を確認する

### 実行コマンド

```sql
WITH rels AS (
  SELECT
    n.nspname AS schema_name,
    c.relname AS table_name,
    c.oid     AS relid
  FROM pg_class c
  JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE c.relkind = 'r'
    AND n.nspname NOT LIKE 'pg_%'
    AND n.nspname <> 'information_schema'
)
SELECT
  row_number() OVER (ORDER BY pg_total_relation_size(relid) DESC) AS id,
  substr(md5(schema_name), 1, 10) AS schema_hash,
  substr(md5(schema_name || '.' || table_name), 1, 10) AS table_hash,
  pg_size_pretty(pg_total_relation_size(relid)) AS total,
  pg_size_pretty(pg_relation_size(relid))       AS heap,
  pg_size_pretty(pg_total_relation_size(relid) - pg_relation_size(relid)) AS others,
  COALESCE((SELECT reltuples::bigint FROM pg_class WHERE oid = relid), 0) AS approx_rows
FROM rels
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 20;
```

### 期待される出力例※研修の出力結果です

```text
 id | schema_hash | table_hash |  total  |  heap   | others  | approx_rows
----+-------------+------------+---------+---------+---------+-------------
  1 | a1b2c3d4e5  | f698d22a22 | 17 GB   | 300 MB  | 17 GB   |     2499935
  2 | a1b2c3d4e5  | f9899d1808 | 2520 kB | 1152 kB | 1368 kB |       19547
  3 | a1b2c3d4e5  | 5314db9deb | 2080 kB | 1792 kB | 288 kB  |        4539
(3 rows)
```

### 出力結果の意味

- `table_hash`
  - テーブル名を匿名化した 10 文字ハッシュ
- `total`
  - テーブル全体の合計サイズ
- `heap`
  - テーブル本体サイズ
- `others`
  - インデックス、TOAST、付随領域を含む合計
- `approx_rows`
  - 統計ベースのおおよその行数

### この研修での読み方

- `f698d22a22` がほぼ全容量を占めている
- `heap = 300 MB` に対して `others = 17 GB` と極端なので、テーブル本体以外が主因の可能性
- 次のステップで `others` の内訳が、インデックスなのか TOAST なのかを確定する

### 次にやること

- 最大テーブルの `heap / indexes / toast_and_other` を確認する

---

## 15. 最大テーブルの容量内訳を確認する

### 目的

- `others` の内訳の細分化
- 移行時間の主因がインデックスか TOAST かを見分け

### 実行コマンド

```sql
WITH target AS (
  SELECT c.oid AS relid
  FROM pg_class c
  JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE c.relkind = 'r'
    AND n.nspname NOT LIKE 'pg_%'
    AND n.nspname <> 'information_schema'
    AND substr(md5(n.nspname || '.' || c.relname), 1, 10) = '<対象テーブルハッシュ>'
)
SELECT
  pg_size_pretty(pg_relation_size(relid)) AS heap,
  pg_size_pretty(pg_indexes_size(relid))  AS indexes,
  pg_size_pretty(pg_total_relation_size(relid)
                 - pg_relation_size(relid)
                 - pg_indexes_size(relid)) AS toast_and_other,
  pg_size_pretty(pg_total_relation_size(relid)) AS total
FROM target;
```

### 変数の意味

- `<対象テーブルハッシュ>`
  - Step 14 で見つけた最大テーブルのハッシュ値
  - 例: `f698d22a22`
  - この値は SQL の文字列なので、**シングルクォートは残したまま**差し替える

### 期待される出力例

```text
  heap  | indexes | toast_and_other | total
--------+---------+-----------------+-------
 300 MB | 208 MB  | 17 GB           | 17 GB
(1 row)
```

### 出力結果の意味

- `heap`
  - テーブル本体はそれほど大きくない
- `indexes`
  - インデックスも主因ではない
- `toast_and_other`
  - 大半が TOAST 側に寄っている可能性が高い

### この研修での読み方

- 17 GB の正体は、インデックス増大ではなく TOAST が主に大きい
- `text` / `json` / `jsonb` / `bytea` などの大きいデータが主因候補

### 次にやること

- TOAST が膨張なのか、実データなのかを確認

---

## 16. TOAST が膨張なのか、実データなのかを確認する

### 目的

- TOAST の大きさがメンテ不足による膨張か、実データによるものかを判断する
- `VACUUM` など、事前作業が必要かどうかの材料を得る

### 実行コマンド

```sql
WITH target AS (
  SELECT c.oid AS main_relid, c.reltoastrelid AS toast_relid
  FROM pg_class c
  JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE c.relkind = 'r'
    AND n.nspname NOT LIKE 'pg_%'
    AND n.nspname <> 'information_schema'
    AND substr(md5(n.nspname || '.' || c.relname), 1, 10) = '<対象テーブルハッシュ>'
)
SELECT
  'main_table' AS which,
  pg_size_pretty(pg_total_relation_size(t.main_relid)) AS total,
  s.n_live_tup,
  s.n_dead_tup,
  s.last_autovacuum,
  s.last_vacuum,
  s.last_autoanalyze,
  s.last_analyze
FROM target t
JOIN pg_stat_all_tables s ON s.relid = t.main_relid

UNION ALL

SELECT
  'toast_table' AS which,
  pg_size_pretty(pg_total_relation_size(t.toast_relid)) AS total,
  s.n_live_tup,
  s.n_dead_tup,
  s.last_autovacuum,
  s.last_vacuum,
  s.last_autoanalyze,
  s.last_analyze
FROM target t
JOIN pg_stat_all_tables s ON s.relid = t.toast_relid;
```

### 変数の意味

- `<対象テーブルハッシュ>`
  - Step 14 で見つけた対象テーブルのハッシュ値

### 期待される出力例

```text
    which    | total | n_live_tup | n_dead_tup |        last_autovacuum        | last_vacuum |       last_autoanalyze        | last_analyze
-------------+-------+------------+------------+-------------------------------+-------------+-------------------------------+--------------
 main_table  | 17 GB |    2499935 |          0 | 2025-02-28 11:17:19.033127+09 |             | 2025-02-28 11:30:34.835884+09 |
 toast_table | 17 GB |     197840 |          0 | 2025-02-28 12:54:41.426003+09 |             |                               |
(2 rows)
```

### 出力結果の意味

- `n_live_tup`
  - 行数の統計値
- `n_dead_tup`
  - 削除済み・更新済みで不要になった行の統計値
- `last_autovacuum`
  - 自動バキュームの最終実行時刻
- `last_autoanalyze`
  - 自動解析の最終実行時刻

### 判定の目安

- `n_dead_tup` が極端に多い場合
  - 膨張の可能性
- `n_dead_tup = 0` かごく少ないのに容量が大きい
  - 実データが大きい可能性

### この案件での読み方

- `main_table` と `toast_table` のどちらも `n_dead_tup = 0`
- したがって、17 GB は膨張ではなく**実データ由来**と判断できる

### 次にやること

- どの列が巨大データの候補か確認する

---

## 17. 巨大値を持つ列候補を匿名化して調べる

### 目的

- `text` や `varchar` など、巨大化の原因になりそうな列を洗い出し
- まずは軽量に `pg_stats` 統計から当たりを付ける

### 実行コマンド

```sql
WITH tgt AS (
  SELECT
    c.oid AS relid,
    n.nspname AS schema_name,
    c.relname AS table_name
  FROM pg_class c
  JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE c.relkind = 'r'
    AND n.nspname NOT LIKE 'pg_%'
    AND n.nspname <> 'information_schema'
    AND substr(md5(n.nspname || '.' || c.relname), 1, 10) = '<対象テーブルハッシュ>'
)
SELECT
  row_number() OVER (ORDER BY s.avg_width DESC NULLS LAST) AS id,
  substr(md5(a.attname), 1, 10) AS col_hash,
  format_type(a.atttypid, a.atttypmod) AS data_type,
  s.avg_width,
  s.null_frac,
  s.n_distinct
FROM tgt
JOIN pg_attribute a ON a.attrelid = tgt.relid
LEFT JOIN pg_stats s
  ON s.schemaname = tgt.schema_name
 AND s.tablename  = tgt.table_name
 AND s.attname    = a.attname
WHERE a.attnum > 0
  AND NOT a.attisdropped
ORDER BY s.avg_width DESC NULLS LAST
LIMIT 20;
```

### 変数の意味

- `<対象テーブルハッシュ>`
  - Step 14 で見つけた対象テーブルのハッシュ値

### 期待される出力例

```text
 id |  col_hash  |           data_type            | avg_width | null_frac | n_distinct
----+------------+--------------------------------+-----------+-----------+------------
  1 | 1111111111 | text                           |        17 |         0 |         -1
  2 | 2222222222 | character varying              |        17 |         0 |         -1
  3 | 3333333333 | timestamp without time zone    |         8 |         0 |         14
  4 | 4444444444 | integer                        |         4 |         0 |         10
(4 rows)
```

### 出力結果の意味

- `col_hash`
  - 列名を匿名化した 10 文字ハッシュ
- `data_type`
  - 列のデータ型
- `avg_width`
  - 統計ベースの平均列幅
- `null_frac`
  - NULL の割合
- `n_distinct`
  - おおよその distinct 数

### 注意

- `avg_width` は統計値なので、巨大値が一部行に偏っている場合は実態より小さく見えることがあリマス
- ここでは**候補列の予測**まで行う

### 次にやること

- サンプル実測で `max / p99 / avg` を確認する

---

## 18. サンプル実測で巨大列を特定する

### 目的

- 統計値ではなく、実データのサンプルから列サイズを確認する
- どの列が TOAST の主因かを具体的に見つける

### 実行コマンド

次の SQL を貼り付けたあと、続けて `\gexec` を実行します。

```sql
WITH target AS (
  SELECT c.oid AS main_relid
  FROM pg_class c
  JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE c.relkind = 'r'
    AND n.nspname NOT LIKE 'pg_%'
    AND n.nspname <> 'information_schema'
    AND substr(md5(n.nspname || '.' || c.relname), 1, 10) = '<対象テーブルハッシュ>'
)
SELECT format($$
SELECT
  '%s' AS col_hash,
  '%s' AS data_type,
  count(*) AS sample_rows,
  round(100.0 * count(*) FILTER (WHERE %I IS NOT NULL) / GREATEST(count(*),1), 2) AS nonnull_pct,
  pg_size_pretty(max(pg_column_size(%I))::bigint) AS max_size,
  pg_size_pretty((percentile_cont(0.99) WITHIN GROUP (ORDER BY pg_column_size(%I)))::bigint) AS p99_size,
  pg_size_pretty(avg(pg_column_size(%I))::bigint) AS avg_size
FROM %s TABLESAMPLE SYSTEM (1);
$$,
  substr(md5(a.attname), 1, 10),
  format_type(a.atttypid, a.atttypmod),
  a.attname,
  a.attname,
  a.attname,
  a.attname,
  t.main_relid::regclass
)
FROM target t
JOIN pg_attribute a ON a.attrelid = t.main_relid
WHERE a.attnum > 0
  AND NOT a.attisdropped
  AND a.atttypid IN ('text'::regtype, 'character varying'::regtype, 'bytea'::regtype, 'json'::regtype, 'jsonb'::regtype)
ORDER BY a.attnum;
```

続けて:

```sql
\gexec
```

### 変数の意味

- `<対象テーブルハッシュ>`
  - Step 14 で見つけた対象テーブルのハッシュ値

### 期待される出力例

```text
  col_hash  |     data_type     | sample_rows | nonnull_pct | max_size | p99_size | avg_size
------------+-------------------+-------------+-------------+----------+----------+----------
 2222222222 | character varying |       26390 |      100.00 | 18 bytes | 18 bytes | 18 bytes
(1 row)

  col_hash  | data_type | sample_rows | nonnull_pct |  max_size  |  p99_size  |  avg_size
------------+-----------+-------------+-------------+------------+------------+------------
 1111111111 | text      |       26390 |      100.00 | 3501 bytes | 3501 bytes | 3501 bytes
(1 row)
```

### 出力結果の意味

- `sample_rows`
  - サンプルに含まれた件数
- `nonnull_pct`
  - NULL でない行の割合
- `max_size`
  - サンプル内の最大サイズ
- `p99_size`
  - サンプルの上位 1% あたりのサイズ感
- `avg_size`
  - サンプル平均サイズ

### この研修での読み方

- `varchar` 列は 18 bytes 程度で主因ではないと判断
- `text` 列が `3501 bytes / nonnull 100% / p99 = max = avg` なので、ほぼ全行に一定サイズの text が格納されている可能性あり
- TOAST が巨大なのは、この `text` 列であると推定

### 注意

- `TABLESAMPLE SYSTEM (1)` は 1% 程度の**ページサンプル**であり、厳密なランダム抽出ではない
- 負荷を下げたい場合は `1` を `0.2` に下げてもよい

### 次にやること

- 対象テーブルの実名を手元で確認し、テーブル定義を参照する

---

## 19. 対象テーブルの実名を手元だけで確認する

### 目的

- 匿名化された `table_hash` が実際にはどのテーブルかを自分だけで特定する
- `\d+` や `pg_dump -t` に使う実テーブル名を把握する

### 実行コマンド

```sql
SELECT
  n.nspname AS schema_name,
  c.relname AS table_name,
  substr(md5(n.nspname || '.' || c.relname), 1, 10) AS table_hash
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r'
  AND n.nspname NOT LIKE 'pg_%'
  AND n.nspname <> 'information_schema'
ORDER BY pg_total_relation_size(c.oid) DESC
LIMIT 50;
```

### 期待される出力例

```text
 schema_name | table_name | table_hash
-------------+------------+------------
 public      | articles   | f698d22a22
 public      | members    | f9899d1808
(2 rows)
```

### 出力結果の意味

- `schema_name.table_name`
  - 実際のテーブル名
- `table_hash`
  - 匿名化に使っていた値

### 注意

- この結果を外部共有行為は禁止です。
- 実名を社外や広い範囲へ共有する場合はマスキング方針を確認

### 次にやること

- `\d+ <調査対象スキーマ名>.<調査対象テーブル名>` で構造を見る

---

## 20. 元テーブルの構造を見る

### 目的

- 列名、型、制約、デフォルト値、インデックスを確認する
- テストでどの程度まで再現するか判定する

### 実行コマンド

```sql
\d+ <調査対象スキーマ名>.<調査対象テーブル名>
```

必要に応じて、インデックス全体も確認する:

```sql
\di+ <調査対象スキーマ名>.*
```

### 変数の意味

- `<調査対象スキーマ名>`
  - 実テーブルが存在するスキーマ名
- `<調査対象テーブル名>`
  - 実際のテーブル名

### 期待される出力例

```text
                                                     Table "public.articles"
   Column   |            Type             | Collation | Nullable |               Default               | Storage  | Stats target | Description
------------+-----------------------------+-----------+----------+-------------------------------------+----------+--------------+-------------
 id         | bigint                      |           | not null | nextval('articles_id_seq'::regclass)| plain    |              |
 title      | character varying(255)      |           | not null |                                     | extended |              |
 body       | text                        |           | not null |                                     | extended |              |
 created_at | timestamp without time zone |           | not null | now()                               | plain    |              |
Indexes:
    "articles_pkey" PRIMARY KEY, btree (id)
```

### 出力結果の意味

- `Column`
  - 列名
- `Type`
  - データ型
- `Nullable`
  - NULL 許可有無
- `Default`
  - デフォルト値
- `Storage = extended`
  - TOAST 対象になりうる列であることを示すことが多い
- `Indexes`
  - 主キーや補助インデックスの情報

### この研修での見方

- `text` 列の有無
- `NOT NULL` の有無
- `DEFAULT` の有無
- 主キー/ユニーク制約の有無
- 依存するシーケンスの有無

### 次にやること

- そのテーブルの DDL を取得する