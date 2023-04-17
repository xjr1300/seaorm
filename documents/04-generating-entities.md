# エンティティの生成

- [エンティティの生成](#エンティティの生成)
  - [`sea-orm-cli`を使用する](#sea-orm-cliを使用する)
    - [環境変数の設定](#環境変数の設定)
    - [ヘルプの入手](#ヘルプの入手)
    - [エンティティファイルの生成](#エンティティファイルの生成)
  - [エンティティ構造体](#エンティティ構造体)
    - [エンティティ](#エンティティ)
      - [テーブル名](#テーブル名)
    - [列](#列)
      - [列名](#列名)
      - [列の型](#列の型)

## `sea-orm-cli`を使用する

最初に、`cargo`で`sea-orm-cli`をインストールします。

```bash
cargo install sea-orm-cli
```

### 環境変数の設定

環境変数に`DATABASE_URL`を設定するか、プロジェクトのルートに`.env`ファイルを作成します。
そして、データベース接続設定を指定します。

```.env
DATABASE_URL=sql://username:password@localhost/database
```

### ヘルプの入手

ヘルプを表示するために、任意のCLIコマンドまたはサブコマンドに対して、`-h`フラグを使用します。

```bash
# 利用可能なすべてのコマンドを表示
sea-orm-cli -h

# `generate`コマンドのすべてのサブコマンドを表示
sea-orm-cli generate -h

# `generate entity`サブコマンドを使用する方法を表示
sea-orm-cli generate entity -h
```

### エンティティファイルの生成

データベース内のすべてのテーブルを発見して、それぞれのテーブルに対応する`SeaORM`のエンティティファイルを生成します。

サポートしているデータベースは次です。

- `MySQL`
- `PostgreSQL`
- `SQLite`

コマンドラインオプションは次です。

- `-u / --database-url`: データベースURL（デフォルトは環境変数に設定された`DATABASE_URL`）
- `-s / --database-schema`: データベーススキーマ（デフォルトは環境変数に設定された`DATABASE_SCHEMA`）
  - `MySQL`と`SQLite`の場合、この引数は無視されます。
  - `PostgreSQL`の場合、この引数はデフォルト値`public`を持つオプションです。
- `-o / --output-dir`: エンティティファイルを出力するディレクトリ（デフォルトはカレントディレクトリ）
- `-v / --verbose`: デバッグメッセージを表示
- `--include-hidden-tables`: 隠しテーブルからエンティティファイルを生成（アンダースコアで名前が始まるテーブルは隠しテーブルで、デフォルトでは無視します）
- `--compact-format`: [コンパクトフォーマット](https://www.sea-ql.org/SeaORM/docs/generate-entity/entity-structure)でエンティティファイルを生成（デフォルトはtrue）
- `--expand-format`: [展開されたフォーマット](https://www.sea-ql.org/SeaORM/docs/generate-entity/expanded-entity-structure)でエンティティファイルを生成
- `--with-serde`: 自動的にエンティティに`serde`の`Serialize`/`Deserialize`トレイトを導出（`none`、`serialize`、`deserialize`、`both`）（デフォルトは`none`）
  - `--serde-skip-deserializing-primary-key`: `#[serde(skip_deserializing)]`でラベルをつけられたプライマリキーフィールドを伴ってエンティティモデルを生成
  - `--serde-skip-hidden-column`: `#[serde(skip)]`でラベルをつけられた隠し列（`_`で始まる列名）を伴ってエンティティモデルを生成
- `--date-time-crate`: エンティティを生成するために使用する日時クレート（`chrono`、`time`）（デフォルトは`chrono`）
- `--max-connections`: コネクションプール内の初期化されたデータベースコネクションの最大数（デフォルトは`1`）
- `--model-extra-derives`: 生成されたモデル構造体に追加する導出マクロ
- `--model-extra-attributes`: 生成されたモデル構造体に追加する属性

```bash
# `bakery`データベースのエンティティファイルを生成して、`src/entity`ディレクトリに出力
sea-orm-cli generate entity -u protocol://username:password@localhost/bakery -o entity/src
```

## エンティティ構造体

単純な[Cake](https://github.com/SeaQL/sea-orm/blob/master/src/tests_cfg/cake.rs)エンティティを確認します。

```rust
use sea_orm::entity::prelude::*;

#[derive(Clone, Debug, PartialEq, Eq, DeriveEntityModel)]
#[sea_orm(table_name = "cake")]
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i32,
    pub name: String,
}

#[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
pub enum Relation {
    #[sea_orm(has_many = "super::fruit::Entity")]
    Fruit,
}

impl Related<super::fruit::Entity> for Entity {
    fn to() -> RelationDef {
        Relation::Fruit.def()
    }
}

impl ActiveModelBehavior for ActiveModel {}
```

> ℹ️ INFO
>
> もし、任意の関連や追加の振る舞いが必要なくても、`Relation`列挙型と`ActiveModelBehavior`のimplブロックを削除しないでください。
> エンティティが十分に機能させるためにそれらのボディを空に保ってください。

### エンティティ

`DeriveEntityModel`マクロは、`Model`、`Column`と`PrimaryKey`を関連付けて`Entity`として定義する面倒な作業をすべて実施します。

#### テーブル名

`table_name`属性は、データベース内で対応するテーブルを指定します。
また、オプションで`schema_name`によってデータベーススキーマまたはデータベース名も指定できます。

```rust
#[sea_orm(table_name = "cake", schema_name = "public")]
pub struct Cake { ... }
```

### 列

#### 列名

すべての列の名前はスネークケースで想定されます。
`column_name`属性を指定することで列名をオーバーライドできます。

```rust
#[sea_orm(column_name = "name")]
pub name: String
```

#### 列の型

列の型は次のマッピングで自動的に導出されます。

Rustの基本的なデータ型のマッピングは次の通りです。

| `Rust`の型 | データベースの型（[ColumnType](https://docs.rs/sea-orm/*/sea_orm/entity/enum.ColumnType.html)） | SQLiteデータ型 | MySQLデータ型 | PostgreSQLデータ型 |
| --- | --- | --- | --- | --- |
| `char` | Char | text | char | char |
| `String` | String | text | varchar | varchar |
| `i8` | TinyInteger | integer | tinyint |char |
| `u8` | TinyUnsigned | integer | tinyint unsigned | N/A |
| `i16` | SmallInteger | integer | smallint | smallint |
| `u16`| SmallUnsigned | integer | smallint unsigned | N/A |
| `i32` | Integer | integer | int | integer |
| `u32` | Unsigned | integer | int unsigned | N/A |
| `i64` | BigInteger | integer | bigint | bigint |
| `u64` | BigUnsigned | N/A | bigint unsigned | N/A |
| `f32` | Float | real | float | real |
| `f64` | Double | real | double | double precision |
| `bool` | Boolean | integer | bool | bool |
| `Vec<u8>` | Binary | blob | blob | bytea |

 Rustの基本的でないデータ型のマッピングについて、すべての再エクスポートされた型を[entity/prelude.rs](https://github.com/SeaQL/sea-orm/blob/master/src/entity/prelude.rs)で確認してください。

| Rust型 | データベース型（[ColumnType](https://docs.rs/sea-orm/*/sea_orm/entity/enum.ColumnType.html)）| SQLiteデータ型 | MySQLデータ型 | PostgreSQLデータ型 |
| --- | --- | --- | --- | --- |
| `Date`: chrono::NaiveDate、`TimeDate`: time::Date | Date | text | date | date |
| `Time`: chrono::NaiveTime、`TimeTime`: time::Time | Time | text | time | time |
| `DateTime`: chrono::NaiveDateTime、`TimeDateTime`: time::PrimitiveDateTime | DateTime | text | datetime | timestamp |
| `DateTimeLocal`: chrono::DateTime<`Local`>、 `DateTimeUtc`: chrono::DateTime<`Utc`> | Timestamp | text | timestamp | N/A |
| `DateTimeWithTimeZone`: chrono::DateTime<`FixedOffset`>、`TimeDateTimeWithTimeZone`: time::OffsetDateTime | TimestampWithTimeZone | text | timestamp | timestamp with time zone |
| `Uuid`: uuid::Uuid, uuid::fmt::Braced, uuid::fmt::Hyphenated, uuid::fmt::Simple, uuid::fmt::Urn | Uuid | text | binary(16) | uuid |
| `Json`: serde_json::Value | Json | text | json | json |
| `Decimal`: rust_decimal::Decimal | Decimal | real | decimal | decimal |

`column_type`属性によって、Rust型と`ColumnType`間のデフォルトのマッピングを上書きできます。

```rust
#[sea_orm(column_type = "Text")]
pub name: String
```

もし、構造体にJSONフィールドをデシリアライズする必要がある場合、そのために`FromJsonQueryResult`を導出する必要があるかもしれません。

```rust
#[derive(Clone, Debug, PartialEq, Eq, DeriveEntityModel)]
#[sea_orm(table_name = "json_struct")]
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i32,
    // `serde_json::Value`で定義されたJSON列
    pub json: Json,
    // カスタム構造体で定義されたJSON列
    pub json_value: KeyValue,
    pub json_value_opt: Option<KeyValue>,
}

// カスタム構造体は、`FromJsonQueryResult`、`Serialize`そして`Deserialize`を導出しなければならない。
#[derive(Clone, Debug, PartialEq, Eq, Serialize, Deserialize, FromJsonQueryResult)]
pub struct KeyValue {
    pub id: i32,
    pub name: String,
    pub price: f32,
    pub notes: Option<String>,
}
```

> ℹ️ 情報
>
> PostgreSQLで配列データ型がサポートされており、モデル内でSeaORMによって既にサポートされている任意の型のベクタを定義できます。
> `postgres-array`フィチャーを有効にする必要があることと、これはPostgresのみの機能であることに注意してください。

```rust
#[derive(Clone, Debug, PartialEq, DeriveEntityModel)]
#[sea_orm(table_name = "collection")]
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i32,
    pub integers: Vec<i32>,
    pub integers_opt: Option<Vec<i32>>,
    pub floats: Vec<f32>,
    pub doubles: Vec<f64>,
    pub strings: Vec<String>,
}
```
