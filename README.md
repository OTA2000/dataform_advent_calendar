[BigQuery Advent Calendar 2020](https://qiita.com/advent-calendar/2020/bigquery)の24日目の記事です。
https://qiita.com/advent-calendar/2020/bigquery

BigQueryのCOVID-19一般公開データセットをもとに、Google Cloudファミリーの一員となった**Dataform**で新規感染者数・死者数の予測誤差を計算するパイプラインを組んでみたので紹介します。

# Dataformとは
https://dataform.co/

Dataformは、従来のETLパイプラインではなく**ELTパイプライン**で役立つツールです。
ELTパイプラインとは、データソースから抽出(**Extract**)した生データをそのままデータウェアハウス(DWH)へ読み込み(**Load**)、DWH内で目的に応じたテーブル構造に変換(**Transform**)する一連の流れのことを指します。Dataformは、ELTパイプラインにおける**Transform**に特化したツールです。

## Google Cloud 傘下に
2020年12月8日(米国時間)、Google CloudがDataformを買収したという[アナウンス](https://cloud.google.com/blog/ja/products/data-analytics/welcoming-dataform-to-bigquery)がありました。同時に、すべてのユーザーが無料でDataformを使用できるようになりました。

>Google Cloud に Dataform を迎え入れることを嬉しく思います。組織全体ですべてのユーザーが分析情報を利用できるようにするという使命を引き続き遂行してまいります。このたび、**すべてのユーザーに Dataform のサービスを無料で提供することにしました**。今後、Dataform と BigQuery の優れた機能を組み合わせて利用できるようになることを楽しみにしています。

# 実際に試してみる
今回のハンズオンで作成したDataformのソースコードはこちらです。
https://github.com/OTA2000/dataform_advent_calendar

## 前提
- ソースデータとして[COVID-19 Public Datasets](https://console.cloud.google.com/marketplace/product/bigquery-public-datasets/covid19-public-data-program)(一般公開データ)を使用します
	- 大抵のユースケースでは事前にBigQueryにソースデータを格納するパイプラインを組む必要があります(今回は省略)

## 今回やったこと
- Dataformプロジェクトの立ち上げ
- ソースデータを宣言するSQLXを作成する
- ビューを定義するSQLXを作成する
- パーティションテーブルを定義するSQLXを作成する
- 通常のテーブルを定義するSQLXを作成する
- 各テーブルの依存関係を確認する
- スケジュールを組んで定時実行させる

## Dataformプロジェクトの立ち上げ
### [GCP] サービスアカウントを作成する
対象のGCPプロジェクトで**サービスアカウント**を作成し**BigQuery管理者**ロールを付与します。
![](https://storage.googleapis.com/zenn-user-upload/3qy03zb75yd2qapsf4gkbopyhlnt)
![](https://storage.googleapis.com/zenn-user-upload/k2oisxzi4p69jjfwrgujpd702yra)
サービスアカウントを作成したらJSONの秘密鍵をダウンロードして保管します。
※Dataformから発行されるBigQueryのジョブはすべてこのサービスアカウントで実行されます。

### [Dataform] ユーザーアカウントを作成する
以下のリンクからDataformのユーザーアカウントを作成します
https://app.dataform.co/

### [Dataform] プロジェクトを作成する
サインアップが完了したら[新規プロジェクトを作成](https://app.dataform.co/#/new)します。
1. 任意のプロジェクト名を入力する
![](https://storage.googleapis.com/zenn-user-upload/dbpi31o4hb52q3xc4btdumnhdmip)
2. 接続対象のGCPプロジェクトIDを入力する
![](https://storage.googleapis.com/zenn-user-upload/6396af2ob12nlvwqheqhz0qjvd6j)
3. データセットを作成するロケーションとサービスアカウントキーを指定してBigQueryと接続する
![](https://storage.googleapis.com/zenn-user-upload/bulh1d0gp3ha6efx5nozlzl67dxi)

### (オプション) 任意のGitプロバイダでバージョン管理する
Dataform上で作成するソースコードはGitでバージョン管理されます。Git管理のプロバイダを任意のもの(GitHub, GitLab, Azure DevOps)に移行することも出来ます。
Gitプロバイダを移行したい場合、左ペインのProject settings > Version controlから設定してください(空のレポジトリとアクセストークンの用意が別途必要)。

## SQLXとは
DataformではSQLXという形式でテーブルやビューを定義します。まずは、SQLXの概要や記法を理解しましょう。
https://docs.dataform.co/introduction/dataform-in-5-minutes
https://docs.dataform.co/guides/sqlx

## ソースデータを宣言するSQLXを作成する
以下のテーブルをソースデータとして使用することをSQLXで宣言します。
- `bigquery-public-data.covid19_open_data.covid19_open_data`
- `bigquery-public-data.covid19_public_forecasts.japan_prefecture_28d`

例: `bigquery-public-data.covid19_open_data.covid19_open_data`を宣言するSQLX
```sql:definitions/declaration/covid19_open_data.sqlx
config {
  type: "declaration",
  database: "bigquery-public-data",
  schema: "covid19_open_data",
  name: "covid19_open_data"
}
```
configブロックの`type`で"declaration"を指定し、読込元のプロジェクトID(database)・データセット名(schema)・テーブル名(name)を定義します。この宣言により、テーブルやビューを作成するSQLXからこのテーブルを呼び出したり、Dataformに依存関係を解釈させたりすることが出来ます。

## ビューを定義するSQLXを作成する
日本全体の日別新規感染者数と死者数を算出するビューをSQLXで定義しBigQuery上に作成します。

```sql:definitions/japan_actual.sqlx
config {
  type: "view",
  schema: "covid19",
  name: "japan_actual",
  description: "都道府県別感染状況(日別)",
  columns: {
    date: "日",
    prefecture_code: "都道府県コード",
    new_confirmed: "新規感染者数",
    new_deaths: "新規死者数"
  }
}

# standardSQL
SELECT
  date,
  REPLACE(location_key, "_", "-") AS prefecture_code,
  SUM(new_confirmed) AS new_confirmed,
  SUM(new_deceased) AS new_deaths
FROM
  ${ref("covid19_open_data")}
WHERE
  country_code = "JP"
  AND location_key <> "JP"
GROUP BY
  date, prefecture_code

```
configブロックで`type`に"view"を指定しビューの説明(description)や各カラムの説明(columns)を定義します。SQLブロックにはビューで使用する標準SQLを記述します。
FROM句で`${ref("covid19_open_data")}`という形で先程宣言したソースデータを呼び出しています。この形式でテーブル参照を記述することでDataformが依存関係を解釈することが出来ます。

![](https://storage.googleapis.com/zenn-user-upload/1hbdfw4phbl4utzy9gwdp0xb6smo)
SQLXを作成すると、右ペインでビューの作成先や依存関係(`bigquery-public-data.covid19_open_data.covid19_open_data`に依存している)、コンパイルされたクエリ(`${ref("covid19_open_data")}`の内容が展開されている)などが表示されます。**PREVIEW RESULTS**でクエリ結果の確認(実際にBigQueryにクエリジョブが発行されるので課金量など要注意)、**CREATE VIEW**でビューの作成が出来ます。

## パーティションテーブルを定義するSQLXを作成する
`bigquery-public-data.covid19_public_forecasts.japan_prefecture_28d`から都道府県別感染状況予測(日別)をパーティションテーブルで作成するためのSQLXを作成します。

```sql:definitions/japan_forecast.sqlx
config {
  type: "incremental",
  schema: "covid19",
  name: "japan_forecast",
  description: "都道府県別感染状況予測(日別)",
  columns: {
    prefecture_code: "都道府県コード",
    prefecture_name_kanji: "都道府県名",
    forecast_date: "予測実施日",
    prediction_date: "予測対象日",
    new_confirmed: "新規感染者数予測",
    new_deaths: "新規死者数予測"
  },
  bigquery: {
    partitionBy: "prediction_date"
  },
  tags: ["daily"]
}

# standardSQL
SELECT
  prefecture_code,
  prefecture_name_kanji,
  forecast_date,  -- 予測実施日
  prediction_date,  -- 予測対象日
  new_confirmed,
  new_deaths,
FROM
  ${ref("japan_prefecture_28d")}
${ when(incremental(), `WHERE forecast_date > (SELECT MAX(forecast_date) FROM ${self()})`) }

```
configブロックで`type`に"incremental"を指定することでパーティションテーブルを作成できます。パーティションの詳細については`bigquery`の中で指定します(`partitionBy`など)。
初回実行時やフルリフレッシュを指定した場合は以下のクエリジョブがBigQueryに発行され新規テーブルが作成されます。
```sql
create or replace table `dataform-advent-calendar.covid19.japan_forecast` partition by prediction_date as

# standardSQL
SELECT
  prefecture_code,
  prefecture_name_kanji,
  forecast_date,  -- 予測実施日
  prediction_date,  -- 予測対象日
  new_confirmed,
  new_deaths,
FROM
  `bigquery-public-data.covid19_public_forecasts.japan_prefecture_28d`

```

二回目以降の実行時は以下のようなクエリジョブが発行されINSERTがおこなわれます。
```sql
insert into `dataform-advent-calendar.covid19.japan_forecast`
(prefecture_code,prefecture_name_kanji,forecast_date,prediction_date,new_confirmed,new_deaths)
select prefecture_code,prefecture_name_kanji,forecast_date,prediction_date,new_confirmed,new_deaths
from (

# standardSQL
SELECT
  prefecture_code,
  prefecture_name_kanji,
  forecast_date,  -- 予測実施日
  prediction_date,  -- 予測対象日
  new_confirmed,
  new_deaths,
FROM
  `bigquery-public-data.covid19_public_forecasts.japan_prefecture_28d`
WHERE forecast_date > (SELECT MAX(forecast_date) FROM `dataform-advent-calendar.covid19.japan_forecast`)) as insertions

```

INSERTの場合、SQLブロックの以下の部分が評価されます。
```
${ when(incremental(), `WHERE forecast_date > (SELECT MAX(forecast_date) FROM ${self()})`) }
```
これは、incremental(INSERT)のときは、`WHERE forecast_date > (SELECT MAX(forecast_date) FROM ${self()})`をクエリに追加するというものです。`${self()}`には定義したパーティションテーブル自身のテーブル指定が入ります。

## 通常のテーブルを定義するSQLXを作成する
実績値のビューと予測値のパーティションテーブルを定義したのでこれらを結合するテーブルを定義して予測値と実績値を比較します。

```sql:definitions/japan_forecast_comparison.sqlx
config {
  type: "table",
  schema: "covid19",
  name: "japan_forecast_comparison",
  description: "予測値評価(日別)",
  columns: {
    date: "日",
    prefecture_name_kanji: "都道府県名",
    forecast_new_confirmed: "新規感染者数(予測値)",
    new_confirmed: "新規感染者数",
    new_confirmed_rmse: "新規感染者数予測誤差",
    forecast_new_deaths: "新規死者数(予測値)",
    new_deaths: "新規死者数",
    new_deaths_rmse: "新規死者数予測誤差",
    latest_forecast_date: "最新の予測実施日"
  },
  tags: ["daily"]
}

# standardSQL
WITH t AS (
  SELECT
    f.prefecture_name_kanji,
    f.prediction_date AS date,
    f.forecast_date,
    MAX (f.forecast_date) OVER (
      PARTITION BY f.prefecture_code,
      f.prediction_date
    ) AS latest_forecast_date,
    j.new_confirmed,
    f.new_confirmed AS forecast_new_confirmed,
    SQRT(AVG(POWER(j.new_confirmed - f.new_confirmed, 2))) AS new_confirmed_rmse,
    j.new_deaths,
    f.new_deaths AS forecast_new_deaths,
    SQRT(AVG(POWER(j.new_deaths - f.new_deaths, 2))) AS new_deaths_rmse,
  FROM
    ${ref("japan_forecast")} f
    LEFT JOIN ${ref("japan_actual")} j ON f.prediction_date = j.date
    AND f.prefecture_code = j.prefecture_code
  GROUP BY
    f.prefecture_name_kanji, f.prediction_date, f.forecast_date, f.prefecture_code, forecast_new_confirmed, new_confirmed, forecast_new_deaths, new_deaths
)
SELECT
  date,
  prefecture_name_kanji,
  new_confirmed,
  forecast_new_confirmed,
  new_confirmed_rmse,
  new_deaths,
  forecast_new_deaths,
  new_deaths_rmse,
  latest_forecast_date
FROM
  t
WHERE
  forecast_date = latest_forecast_date

```
このテーブル作成を実行すると以下のようなテーブルが作成されます。
![](https://storage.googleapis.com/zenn-user-upload/9nqensx4if0pg8eb7q2vdskpck94)

## 各テーブルの依存関係を確認する
ハンバーガーメニューからDependency Treeを開くと、定義したテーブルやビューの依存関係を確認できます。
![](https://storage.googleapis.com/zenn-user-upload/vqdzdb8mla991swd49j98bw9boz6)
各ソーステーブルから、実績値のビュー・予測値のパーティションテーブルが作られ、予測比較のテーブルへとつながっていることが分かります。

## スケジュールを組んで定時実行させる
毎朝9時(JST)に予測値のパーティションテーブルと予測誤差のテーブルを更新したいのでスケジュールを組みます。
ファイル一覧から`environments.json`を開くと以下のような画面が表示されます(View as plain textのトグルを有効にするとjsonを直に編集することも出来る)。
![](https://storage.googleapis.com/zenn-user-upload/wsf3f6dck07dxzwa9xjg88ems695)

**CREATE NEW SCHEDULE**から対象のタグや実行時に依存関係を含めて実行するかなどを指定します。
![](https://storage.googleapis.com/zenn-user-upload/f20899x1nu2ihx76mpnbmwzmhpf8)
- スケジュールはcron形式で表記する
- `Tags to run`で定時実行対象のタグ名を入力する
	- 事前にjapan_forecast.sqlxとjapan_forecast_comparison.sqlxのconfigで`tags: ["daily"]`を指定済み
- 実行時にテーブルをフルリフレッシュ(作り直し)するか選択する
- 依存関係を含んで実行するか選択する
- (事前にメールやSlackの通知設定を作成している場合)どのチャネルに通知を飛ばすか選択する
	- 「成功のみ」「失敗のみ」「すべて」の3パターンから選ぶことが出来ます。

# おわりに
データパイプラインをスケジュールクエリで頑張っている方も少なくないかと思います。スケジュールクエリは依存関係などを考慮することが出来ません(実行タイミングをズラすなどするしかない)。Cloud Composerなどのワークフローエンジンを導入していない方は導入しない手はないかと思います。
また、DataformのパイプラインはAPIで呼び出すことも可能なので、ワークフローエンジンを導入している場合でも一連の流れに組み込むことが出来ます。

本稿では触れられませんでしたが、データ品質管理(アサーション)やクエリの単体テストなど充実した機能がまだまだあります。年末年始、Dataformを試して業務に活かしてみてはいかがでしょうか。
