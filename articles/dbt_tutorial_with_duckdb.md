---
title: "ローカル環境のみ利用したdbtチュートリアル"
emoji: "💁"
type: "tech"
topics: ["dbt", "duckdb"]
published: false
---

:::message
基本的には公式チュートリアルを進めるのをお勧めします。
:::

dbtの公式チュートリアルはBigQueryなどクラウド環境を利用する前提で書かれていたので、ローカル環境だけでできるようにDuckDBを用いたチュートリアルを書きました。また、公式チュートリアルにはgitの操作なども含まれていたため、dbtを利用するのに最低限必要そうなもののみに絞って書こうと思います。

以下の記事を参考にしています。
* 公式チュートリアル [Getting started with dbt Core | dbt Developer Hub](https://docs.getdbt.com/docs/get-started/getting-started-dbt-core)
* [DuckDBとdbtとRillで作るローカルで動くDWHっぽいもの](https://zenn.dev/takimo/articles/bb11eab78232f4)


## dbtとは？

dbtはETL（Extract/Transform/Load）でいうところのtransformationを担うツールです。
https://docs.getdbt.com/docs/introduction


## dbtを利用することのメリット

- 変数のようなものを利用できて、同じようなSQLを書かなくていい
- テーブル間の依存関係を表すグラフ（lineage）をSQLから生成することができる
- yamlにテーブル定義書を記述すため、テーブル定義書をgit管理できる
- SQLのテストが簡単にできる

ここら辺がメリットかなと個人的に思います。よく利用するSQLのsnippetをmacro（[Jinja and macros | dbt Developer Hub](https://docs.getdbt.com/docs/build/jinja-macros#macros)）として登録して、関数のように利用できる機能がありますが、SQLが複雑化し管理がしづらいため、利用はなるべく避けるのがいい気がします。


## DuckDBのインストール

ローカルでDB操作ができるようにDuckDBをインストールします。
https://duckdb.org/docs/installation/index
```
curl -OL https://github.com/duckdb/duckdb/releases/download/v0.6.1/duckdb_cli-osx-universal.zip
unzip duckdb_cli-osx-universal.zip
```

### DuckDBの動作確認

```
./duckdb tutorial.db
v0.6.1 919cad22e8
Enter ".help" for usage hints.
D select 'aaaa';
┌─────────┐
│ 'aaaa'  │
│ varchar │
├─────────┤
│ aaaa    │
└─────────┘
D .exit
```

`.help`でヘルプを表、`.exit`でDuckDBのCLIから抜けられます。


## dbtのインストール

今回は、DuckDBを利用するのでdbt-coreとdbt-duckdbをインストールします。BigQueryやSnowflakeなど、それぞれDBごとにライブラリが存在するので、DuckDB以外を利用する場合はDBにあったライブラリをインストールしてください。
```
pip install dbt-core dbt-duckdb
```


## dbtプロジェクト初期セットアップ

`dbt init 適当な名前`を実行して、DBに`duckdb`を選択。

```
dbt init tutorial
09:32:44  Running with dbt=1.3.1
Which database would you like to use?
[1] duckdb

(Don't see the one you want? https://docs.getdbt.com/docs/available-adapters)

Enter a number: 1
09:33:00  No sample profile found for duckdb.
09:33:00
Your new dbt project "tutorial" was created!

For more information on how to configure the profiles.yml file,
please consult the dbt documentation here:

  https://docs.getdbt.com/docs/configure-your-profile

One more thing:

Need help? Don't hesitate to reach out to us via GitHub issues or on Slack:

  https://community.getdbt.com/

Happy modeling!
```

上記のコマンドを実行後、tutorialという名前のディレクトリとtutorialディレクトリの中に、以下のディレクトリが自動生成されます。
```
.
├── analyses
├── macros
├── models
│   └── example
├── seeds
├── snapshots
└── tests
```

それぞれのディレクトリの役割は、[About dbt projects | dbt Developer Hub](https://docs.getdbt.com/docs/build/projects)に記載されています。

| Resource | Description |
|---|---|
| models | 基本的なSQLを置く場所。ここのディレクトリ以下のSQLは`dbt run`を実行した際に実行される。 |
| snapshots | 状態があるようなデータのSQLを置いておく場所。例は[Snapshots \| dbt Developer Hub](https://docs.getdbt.com/docs/build/snapshots)がわかりやすい。 |
| seeds | csvなどの静的ファイルを置いておく場所。 `dbt seed`コマンドでdbtにロードできる。 |
| tests | SQLのテストクエリを置いておく場所。 |
| macros | macroを置いておく場所。 |
| analysis | より分析っぽいアドホックなSQLを置く場所。ここに配置されたSQLは、`dbt run`時、変数置換は行われるが実行はされない。 |

`~/.dbt/profiles.yml` に以下を記載。

```yaml
tutorial:
  outputs:
   dev:
     type: duckdb
     path: 先ほど作成したtutorial.dbまでのパス
  target: dev
```


## データ登録

以下のデータをダウンロードし、 `seeds` ディレクトリに設置。
https://github.com/dbt-labs/jaffle_shop/tree/main/seeds
```
cd seeds
curl -OL https://raw.githubusercontent.com/dbt-labs/jaffle_shop/main/seeds/raw_customers.csv
curl -OL https://raw.githubusercontent.com/dbt-labs/jaffle_shop/main/seeds/raw_orders.csv
```

`dbt seed`コマンドでcsvをDBに登録。

![](/images/dbt_tutorial_with_duckdb/dbt_seed.png)

:::details 実行結果テキスト
```
dbt seed
04:21:05  Running with dbt=1.3.1
04:21:05  Partial parse save file not found. Starting full parse.
04:21:06  Found 3 models, 4 tests, 0 snapshots, 0 analyses, 292 macros, 0 operations, 2 seed files, 0 sources, 0 exposures, 0 metrics
04:21:06
04:21:06  Concurrency: 1 threads (target='dev')
04:21:06
04:21:06  1 of 2 START seed file main.raw_customers ...................................... [RUN]
04:21:06  1 of 2 OK loaded seed file main.raw_customers .................................. [INSERT 100 in 0.11s]
04:21:06  2 of 2 START seed file main.raw_orders ......................................... [RUN]
04:21:06  2 of 2 OK loaded seed file main.raw_orders ..................................... [INSERT 99 in 0.04s]
04:21:06
04:21:06  Finished running 2 seeds in 0 hours 0 minutes and 0.25 seconds (0.25s).
04:21:06
04:21:06  Completed successfully
04:21:06
04:21:06  Done. PASS=2 WARN=0 ERROR=0 SKIP=0 TOTAL=2
```
:::

`tutorial.db`のあるディレクトリまで移動して、テーブルが保存されているか確認しましょう。先ほどと同様に、`./duckdb tutorial.db`でDuckDBを起動し、`.table`コマンドを実行しましょう。登録されているテーブル一覧が表示されます。`select * from raw_customers limit 5;`でcsvから登録されたデータを確認しましょう。

![](/images/dbt_tutorial_with_duckdb/check_table.png)

## Transformを行う（モデルをビルドする）

dbtを利用してtransformを行います。まず、以下のクエリを`models/`以下に、 `customers.sql`として保存してください。

```sql
with customers as (

    select
        id as customer_id,
        first_name,
        last_name

    from {{ ref('raw_customers') }}

),

orders as (

    select
        id as order_id,
        user_id as customer_id,
        order_date,
        status

    from {{ ref('raw_orders') }}

),

customer_orders as (

    select
        customer_id,

        min(order_date) as first_order_date,
        max(order_date) as most_recent_order_date,
        count(order_id) as number_of_orders

    from orders

    group by 1

),

final as (

    select
        customers.customer_id,
        customers.first_name,
        customers.last_name,
        customer_orders.first_order_date,
        customer_orders.most_recent_order_date,
        coalesce(customer_orders.number_of_orders, 0) as number_of_orders

    from customers

    left join customer_orders using (customer_id)

)

select * from final
```

dbtプロジェクトのディレクトリ以下で、dbt runで上記のSQLが実行されtransformができます。

![](/images/dbt_tutorial_with_duckdb/dbt_run.png)

（`models/example`以下にSQLが存在しているため、3つSQLが実行されています。）

:::details 実行結果テキスト
```
dbt run
04:30:32  Running with dbt=1.3.1
04:30:32  Found 3 models, 4 tests, 0 snapshots, 0 analyses, 292 macros, 0 operations, 2 seed files, 0 sources, 0 exposures, 0 metrics
04:30:32
04:30:32  Concurrency: 1 threads (target='dev')
04:30:32
04:30:32  1 of 3 START sql view model main.custoemrs ..................................... [RUN]
04:30:32  1 of 3 OK created sql view model main.custoemrs ................................ [OK in 0.09s]
04:30:32  2 of 3 START sql table model main.my_first_dbt_model ........................... [RUN]
04:30:32  2 of 3 OK created sql table model main.my_first_dbt_model ...................... [OK in 0.06s]
04:30:32  3 of 3 START sql view model main.my_second_dbt_model ........................... [RUN]
04:30:32  3 of 3 OK created sql view model main.my_second_dbt_model ...................... [OK in 0.03s]
04:30:32
04:30:32  Finished running 2 view models, 1 table model in 0 hours 0 minutes and 0.28 seconds (0.28s).
04:30:32
04:30:32  Completed successfully
04:30:32
04:30:32  Done. PASS=3 WARN=0 ERROR=0 SKIP=0 TOTAL=3
```
:::

`{{ ref('raw_customers') }}`などが実体に置き換えられたSQLは`/target/compiled/tutorial/models`以下に格納されています。

## exampleのSQLを削除する

`models/example`にexampleのsqlファイルやyamlファイルがあり必要ないので、削除しておきます。削除後`dbt_project.yml`の末尾に記載の

```yaml
models:
  tutorial:
    # Config indicated by + and applies to all files under models/example/
    example:
      +materialized: view
```

を

```yaml
models:
  tutorial:
    +materialized: table
```

のように書き換えます。

## データとロジックを分離したTransformを行う（モデルの上にモデルをビルドする）

「データをクレンジングするロジック と データを変換するロジックは分ける」というSQLのbest practiceになるように、先程のSQLを書き換えます。

`models/stg_customers.sql`として以下を保存。

```sql
select
    id as customer_id,
    first_name,
    last_name

from {{ ref('raw_customers') }}
```

`models/stg_orders.sql`として以下を保存。

```sql
select
    id as order_id,
    user_id as customer_id,
    order_date,
    status

from {{ ref('raw_orders') }}
```

`models/customers.sql`を以下に書き換えます。

```sql
with customers as (

    select * from {{ ref('stg_customers') }}

),

orders as (

    select * from {{ ref('stg_orders') }}

),

customer_orders as (

    select
        customer_id,

        min(order_date) as first_order_date,
        max(order_date) as most_recent_order_date,
        count(order_id) as number_of_orders

    from orders

    group by 1

),

final as (

    select
        customers.customer_id,
        customers.first_name,
        customers.last_name,
        customer_orders.first_order_date,
        customer_orders.most_recent_order_date,
        coalesce(customer_orders.number_of_orders, 0) as number_of_orders

    from customers

    left join customer_orders using (customer_id)

)

select * from final
```

`dbt run`を実行。`stg_customers`と`stg_orders`と`customers`は別々のテーブルとして作成されます。dbtが自動で実行中順序を推測して順番にクエリが実行されます。`customers`は`stg_customers`と`stg_orders`に依存しているため`customers`が最後に実行されています。


## テストを実行する

dbtでは、`models`ディレクトリにあるSQLにテストを行うことができます。`models/schema.yml`として以下を保存します。

```yaml
version: 2

models:
  - name: customers
    columns:
      - name: customer_id
        tests:
          - unique
          - not_null

  - name: stg_customers
    columns:
      - name: customer_id
        tests:
          - unique
          - not_null

  - name: stg_orders
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: status
        tests:
          - accepted_values:
              values: ['placed', 'shipped', 'completed', 'return_pending', 'returned']
      - name: customer_id
        tests:
          - not_null
          - relationships:
              to: ref('stg_customers')
              field: customer_id
```

`dbt test`でテストを実行することができます。dbtにデフォルトであるテストを実行したり、自作のカスタムテストを各テーブルの各カラムに対して実行することができます。

今回の場合は、以下の4種類のテストが`models/schema.yml`に記載されています。

- unique: ユニークであるか
- not_null: nullが含まれていないか
- accepted_values: 指定した許容値のみかどうか
- relationships: テーブル間のリレーション（今回の例では、`orders`テーブル内の`customer_id`は、`customers`テーブルの`id`に含まれている）

## ドキメントを生成する

先ほど作成した`models/schema.yml`をもとにドキュメントを作成することができます。`models/schema.yml`を以下に書き換えます。

```yaml
version: 2

models:
  - name: customers
    description: One record per customer
    columns:
      - name: customer_id
        description: Primary key
        tests:
          - unique
          - not_null
      - name: first_order_date
        description: NULL when a customer has not yet placed an order.

  - name: stg_customers
    description: This model cleans up customer data
    columns:
      - name: customer_id
        description: Primary key
        tests:
          - unique
          - not_null

  - name: stg_orders
    description: This model cleans up order data
    columns:
      - name: order_id
        description: Primary key
        tests:
          - unique
          - not_null
      - name: status
        tests:
          - accepted_values:
              values: ['placed', 'shipped', 'completed', 'return_pending', 'returned']
```

`dbt docs generate`でドキュメントを生成。`dbt docs serve`でWebUIのドキュメントを起動できます（自動的にページが立ち上がらなかった場合、`http://localhost:8080`にChromeなどのブラウザでアクセスしてください）。画像のような`models/schema.yml`に記載した情報が記載されています。

![トップページ](/images/dbt_tutorial_with_duckdb/docs_overview.png)
*トップページ*

![customersテーブルのドキュメント](/images/dbt_tutorial_with_duckdb/docs_customers_table.png)
*customersテーブルのドキュメント*

![Lineageと呼ばれるテーブルの依存関係図](/images/dbt_tutorial_with_duckdb/docs_customers_table_lineage.png)
*Lineageと呼ばれるテーブルの依存関係図*


以上でチュートリアルは終了です。dbtの公式チュートリアルに則った基本的な使い方のみ紹介したので、他のテストはどうやるのかや、変数を利用したSQLの書き方などは別途、調べてください。dbtを使って分析に必要なデータを作成した後に、PythonからDuckDBに接続して予測モデルとかに入力するとかの実装ができるといいなと思っています。


* 参考（再掲）
  * 公式チュートリアル [Getting started with dbt Core | dbt Developer Hub](https://docs.getdbt.com/docs/get-started/getting-started-dbt-core)
  * [DuckDBとdbtとRillで作るローカルで動くDWHっぽいもの](https://zenn.dev/takimo/articles/bb11eab78232f4)
