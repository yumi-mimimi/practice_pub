# PostgreSQL 旧サーバ調査・移行準備手順書

## 1. 目的

この手順書は、旧サーバー上の PostgreSQL を**変更せずに調査**し、リプレイス/移行に必要な情報を安全に収集するためのものです。

主な目的は次の3つです。

1. 旧DBの構成を把握する
2. 移行先で再現すべき設定を特定する
3. テスト用のDDL/ダミーデータを用意できる状態にする

---

## 2. 安全方針

旧サーバー上では、以下を徹底します。

- **調査は読み取り専用セッションで行う**
- **個人情報や固有名は外部共有時に伏せる**
- 以下のような変更系コマンドは実行しない
  - `INSERT`
  - `UPDATE`
  - `DELETE`
  - `ALTER`
  - `DROP`
  - `TRUNCATE`
  - `VACUUM FULL`
  - `REINDEX`
  - `CREATE TABLE`（旧サーバー上では行わない）

> 注意: `SET default_transaction_read_only = on;` は**その接続セッションだけ**に効きます。  
> `\c` で別DBへつなぎ直したら、**再度設定し直す**こと。

---

## 3. 前提

- EC2 に SSH 接続できること
- PostgreSQL に `psql` で接続できること
- 調査対象は旧サーバー側 PostgreSQL
- テスト用の作業は新サーバーまたは別環境で行うこと

---

## 4. 旧サーバーへ入り、psql を起動する

```bash
sudo -iu postgres
psql
```

接続確認:

```sql
\conninfo
```

例:

```text
You are connected to database "postgres" as user "postgres" via socket in "/tmp" at port "5432".
```

---

## 5. 調査セッションを読み取り専用にする

まず、現在のセッションを読み取り専用に固定します。

これにより意図しない編集を予防できるはずです

```sql
SHOW default_transaction_read_only;
SET default_transaction_read_only = on;
SHOW default_transaction_read_only;
```

出力結果は↓これ:

```text
default_transaction_read_only
-------------------------------
 on
```

---

## 6. DB一覧を匿名化して確認する

DB名をそのまま出したくない場合は、ハッシュ化して一覧化します。

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

### 目的

- DBの特定する
- ストレージの状況を把握する

### 今回の演習結果は↓これ

- DBは実質2つ
- 本命DBは **17 GB**
- もう一方は **約 7.7 MB**

---

## 7. 本命DBの実名を手元だけで確認する

共有しない前提で、DB名とハッシュの対応表を確認します。

```sql
SELECT datname, substr(md5(datname), 1, 10) AS db_name_hash
FROM pg_database
WHERE datname NOT IN ('template0','template1')
ORDER BY pg_database_size(datname) DESC;
```

対応が分かったら、本命DBへ接続します。

```sql
\c <TARGET_DB>
\conninfo
```

> 注意: `\c` の後は読み取り専用設定がリセットされることがあるため、次を再実行すること。

```sql
SET default_transaction_read_only = on;
SHOW default_transaction_read_only;
```

---

## 8. バージョン・文字コード・照合順序を確認する

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

### 今回の確認結果

- PostgreSQL: **11.2**
- Encoding: **UTF8**
- Collate: **en_US.UTF-8**
- CType: **en_US.UTF-8**

### 目的

移行先で以下を再現するため:

- DB作成時の locale
- エンコーディング
- バージョン互換の確認

---

## 9. 拡張機能を確認する

```sql
SELECT extname, extversion, nspname AS schema
FROM pg_extension e
JOIN pg_namespace n ON n.oid = e.extnamespace
ORDER BY extname;
```

### 今回の確認結果

- `plpgsql` のみ

### 目的

移行先で事前導入が必要な拡張の有無を確認するため。

---

## 10. スキーマ構成を匿名化して確認する

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

### 今回の確認結果

- 非システムスキーマ: **1つ**
- テーブル: **42**
- ビュー: **0**
- シーケンス: **38**

---

## 11. 容量の大きいテーブルを匿名化して確認する

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

### 今回の確認結果

最大テーブルの特徴:

- `table_hash = f698d22a22`
- total: **17 GB**
- heap: **300 MB**
- others: **17 GB**
- approx_rows: **2,499,935**

### 読み方

- `heap`: テーブル本体
- `others`: インデックス + TOAST + その他

---

## 12. 巨大テーブルの内訳を確認する

特定テーブルの `heap / indexes / toast` の内訳を確認します。

```sql
WITH target AS (
  SELECT c.oid AS relid
  FROM pg_class c
  JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE c.relkind = 'r'
    AND n.nspname NOT LIKE 'pg_%'
    AND n.nspname <> 'information_schema'
    AND substr(md5(n.nspname || '.' || c.relname), 1, 10) = 'f698d22a22'
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

### 今回の確認結果

- heap: **300 MB**
- indexes: **208 MB**
- toast_and_other: **17 GB**
- total: **17 GB**

### 解釈

このテーブルは**インデックスが巨大なのではなく、TOAST 側が巨大**。
つまり、`text / json / bytea` などの大きい値が実データとして格納されている可能性が高い。

---

## 13. TOAST が膨張なのか、実データなのかを確認する

```sql
WITH target AS (
  SELECT c.oid AS main_relid, c.reltoastrelid AS toast_relid
  FROM pg_class c
  JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE c.relkind = 'r'
    AND n.nspname NOT LIKE 'pg_%'
    AND n.nspname <> 'information_schema'
    AND substr(md5(n.nspname || '.' || c.relname), 1, 10) = 'f698d22a22'
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

### 今回の確認結果

- `main_table`: `n_dead_tup = 0`
- `toast_table`: `n_dead_tup = 0`

### 解釈

TOAST 17 GB は**膨張ではなく実データ**と判断できる。

---

## 14. 巨大値を持つ列を匿名化して調べる

まず、`pg_stats` ベースで候補列を確認します。

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
    AND substr(md5(n.nspname || '.' || c.relname), 1, 10) = 'f698d22a22'
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

### 今回の確認結果

- `text` 列あり
- `character varying` 列あり
- `avg_width` だけでは巨大TOASTの原因は確定しづらい

> `pg_stats.avg_width` は統計値なので、巨大値が一部行に偏っている場合は小さく見えることがある。

---

## 15. サンプル実測で巨大列を特定する

下記を貼り付けたあと、続けて `\gexec` を実行します。

```sql
WITH target AS (
  SELECT c.oid AS main_relid
  FROM pg_class c
  JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE c.relkind = 'r'
    AND n.nspname NOT LIKE 'pg_%'
    AND n.nspname <> 'information_schema'
    AND substr(md5(n.nspname || '.' || c.relname), 1, 10) = 'f698d22a22'
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

### 今回の確認結果

- `character varying`: 約 **18 bytes** 固定
- `text`: **3,501 bytes**、非NULL **100%**、p99/max/avg が同値

### 解釈

巨大TOASTの主因は、**ほぼ全行に格納されている固定長に近い `text` データ**である可能性が高い。

---

## 16. テーブル名の実名を確認する（手元のみ）

巨大テーブルの実名を知りたい場合:

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

この一覧から `table_hash = f698d22a22` に対応する `schema_name.table_name` を控える。

> 外部共有する場合は、実名をそのまま貼らないこと。

---

## 17. テーブル構造を見る

実名が分かったら、テーブル定義を確認します。

```sql
\d+ schema_name.table_name
```

確認できる内容:

- 列名
- データ型
- NULL制約
- デフォルト値
- 主キー/制約
- ストレージ情報
- インデックス

必要に応じて、インデックス一覧も確認します。

```sql
\di+ schema_name.*
```

---

## 18. DDL を抜き出す

テスト環境で同じテーブルを再現したい場合は、DDL を取得します。

psql の外で実行:

```bash
pg_dump -s -d <TARGET_DB> -t 'schema_name.table_name' > table_ddl.sql
```

スキーマ全体の定義だけ必要な場合:

```bash
pg_dump -s -d <TARGET_DB> > schema_only.sql
```

---

## 19. テスト用に同等データを作る方法

### 19-1. 最小構成で巨大TOASTを再現する

新サーバーまたはテストDB上で実行すること。

```sql
CREATE TABLE toast_test (
  id bigserial PRIMARY KEY,
  payload text NOT NULL
);

INSERT INTO toast_test(payload)
SELECT repeat('x', 3501)
FROM generate_series(1, 100000);

ANALYZE toast_test;

SELECT
  pg_size_pretty(pg_relation_size('toast_test')) AS heap,
  pg_size_pretty(pg_indexes_size('toast_test')) AS indexes,
  pg_size_pretty(
    pg_total_relation_size('toast_test')
    - pg_relation_size('toast_test')
    - pg_indexes_size('toast_test')
  ) AS toast,
  pg_size_pretty(pg_total_relation_size('toast_test')) AS total;
```

### 19-2. 元テーブル定義をそのまま複製する

**旧サーバーではなく、テスト環境で**実行すること。

```sql
CREATE TABLE test_schema.test_table (
  LIKE schema_name.table_name INCLUDING ALL
);
```

> `INCLUDING ALL` により、列定義・デフォルト・制約・インデックス等を含めた複製が可能。

---

## 20. 今回の作業で分かったこと（要約）

### 環境情報

- PostgreSQL 11.2
- Encoding: UTF8
- Collate / CType: en_US.UTF-8
- 拡張: `plpgsql` のみ

### 構成情報

- 本命DBサイズ: 17 GB
- 非システムスキーマ: 1
- テーブル数: 42
- シーケンス数: 38

### 容量の主因

- 最大テーブルは約 17 GB
- heap は 300 MB 程度
- indexes は 208 MB 程度
- TOAST が約 17 GB
- dead tuple は 0 のため、膨張ではなく実データ
- `text` 列に 3,501 bytes 程度の値が全行に近く格納されている可能性が高い

### 移行時の示唆

- 時間がかかるのはインデックス再構築よりも、**巨大 `text` データの搬送**
- dump/restore の見積もりは、テーブル本体より **TOAST データ量** を基準に考える
- テスト環境では、まず 10万件程度で再現し、必要なら段階的に件数を増やす

---

## 21. 次にやること

1. 新サーバー側の PostgreSQL バージョンを決める
2. 旧DBと同じ locale / encoding で新DBを作る
3. DDL を取得する
4. テストデータで restore 時間を測る
5. 本番移行方式を決める
   - `pg_dump / pg_restore`
   - 論理レプリケーション
   - 段階移行

---

## 22. 補足

- 旧サーバー側では、今後も**読み取り専用での調査**を継続すること
- `\c` したら読み取り専用をかけ直すこと
- 画面共有やドキュメント化の際は、DB名・スキーマ名・テーブル名・列名を必要に応じてマスクすること

