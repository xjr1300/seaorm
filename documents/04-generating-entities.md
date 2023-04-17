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
      - [追加のプロパティ](#追加のプロパティ)
      - [属性の無視](#属性の無視)
      - [選択と保存で列の型をキャストする](#選択と保存で列の型をキャストする)
    - [プライマリーキー](#プライマリーキー)
      - [自動インクリメント](#自動インクリメント)
      - [複合キー](#複合キー)
    - [関連](#関連)
    - [アクティブモデルの振る舞い](#アクティブモデルの振る舞い)
  - [エンティティ構造体の展開](#エンティティ構造体の展開)
    - [エンティティ](#エンティティ-1)
    - [列](#列-1)
      - [追加の属性](#追加の属性)
      - [選択と保存における列型のキャスト](#選択と保存における列型のキャスト)
    - [プライマリーキー](#プライマリーキー-1)
    - [モデル](#モデル)
      - [NULL許可属性](#null許可属性)
    - [アクティブモデル](#アクティブモデル)
    - [関連](#関連-1)

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

#### 追加のプロパティ

`default_value`、`unique`、`indexed`そして`nullable`の追加属性を列に追加できます。

オプションの属性にカスタムな`column_type`を指定した場合、`nullable`も指定しなければなりません。

#### 属性の無視

データベースのどの列にもマッピングしないような特定のモデルの属性を無視する場合、`ignore`注釈を使用できます。

```rust
#[sea_orm(ignore)]
pub ignore_me: String
```

#### 選択と保存で列の型をキャストする

もし、ある型で列を選択するが、別の方でデータベースに保存する必要がある場合、キャストを実行するために`select_as`と`save_as`属性を記述できます。
典型的なユースケースは、`citext`（テキストの大文字小文字を無視）型の列をRustの`String`で選択して、`citext`としてデータベースに保存することです。
モデルフィールドを下のように定義する必要があります。

```rust
#[sea_orm(select_as = "text", save_as = "citext")]
pub case_insensitive_text: String
```

### プライマリーキー

プライマリーキーとして列をマークするために`primary_key`属性を使用します。

```rust
#[sea_orm(primary_key)]
pub id: i32
```

#### 自動インクリメント

デフォルトで`auto_increment`は`primary_key`列に暗黙的にマークされます。
`false`でそれを上書きします。

```rust
#[sea_orm(primary_key, auto_increment = false)]
pub id: i32
```

#### 複合キー

2つの列のタプルをプライマリーキーとして使用することは、連想（結合、中間）テーブル (junction table)では普通です。
単に、複合プライマリキーを定義するために複数の列に注釈します。
複合キーの場合、デフォルトで`auto_increment`は`false`です。

```rust
pub struct Model {
    #[sea_orm(primary_key)]
    pub cake_id: i32,
    #[sea_orm(primary_key)]
    pub fruit_id: i32,
}
```

### 関連

`DeriveRelation`は、[RelationTrait](https://docs.rs/sea-orm/*/sea_orm/entity/trait.RelationTrait.html)の実装を支援するマクロです。

```rust
#[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
pub enum Relation {
    #[sea_orm(has_many = "super::fruit::Entity")]
    Fruit,
}
```

もし、関連がないのであれば、単純に記述できます。

```rust
#[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
pub enum Relation {}
```

[Related](https://docs.rs/sea-orm/*/sea_orm/entity/trait.Related.html)トレイトはエンティティ同士を接続するため、両方のエンティティを選択するクエリを構築できます。

関連の詳細については、[Relation](https://www.sea-ql.org/SeaORM/docs/relation/one-to-one/)の章を参照してください。

### アクティブモデルの振る舞い

例えば、カスタム検証ロジックを実行、または副作用をトリガーする、`ActiveModel`の様々なアクションのハンドラーです。
トランザクション内で、それが実行された後、アクションを中断でき、データベースにそれを保存することを回避します。

```rust
#[async_trait]
impl ActiveModelBehavior for ActiveModel {
    /// デフォルト値で新しいActionModelを作成します。
    /// また、`Default::default()`によって使用されます。
    fn new() -> Self {
        Self {
            uuid: Set(Uuid::new_v4()),
            ..ActiveModelTrait::default()
        }
    }

    /// 挿入または更新する前にトリガーされます。
    async fn before_save<C>(self, db: &C, insert: bool) -> Result<Self, DbErr>
    where
        C: ConnectionTrait,
    {
        if self.price.as_ref() <= &0.0 {
            Err(DbErr::Custom(format!(
                "[before_save] Invalid Price, insert: {}",
                insert
            )))
        } else {
            Ok(self)
        }
    }

    /// 挿入または更新された後でトリガーされます。
    async fn after_save<C>(model: Model, db: &C, insert: bool) -> Result<Model, DbErr>
    where
        C: ConnectionTrait,
    {
        Ok(model)
    }

    /// 削除する前にトリガーされます。
    async fn before_delete<C>(self, db: &C) -> Result<Self, DbErr>
    where
        C: ConnectionTrait,
    {
        Ok(self)
    }

    /// 削除した後にトリガーされます。
    async fn after_delete<C>(self, db: &C) -> Result<Self, DbErr>
    where
        C: ConnectionTrait,
    {
        Ok(self)
    }
}
```

もし、カスタマイズが必要ないのであれば、簡単に記述します。

```rust
impl ActiveModelBehavior for ActiveModel {}
```

## エンティティ構造体の展開

SeaORMは動的で、それはランタイムで何らかを設定する柔軟性があることを意味しています。
もし、`DeriveEntityModel`が展開する内容に興味があるのであれば、読み続けてください。
そうでないなら、今のところ、これをスキップできます。

展開されたエンティティの形式は、`--expand-format`を付けた`sea-orm-cli`によって生成されます。

[Cake](https://github.com/SeaQL/sea-orm/blob/master/src/tests_cfg/cake_expanded.rs)エンティティの拡張の節に進みます。

### エンティティ

[EntityTrait]を実装することで、与えたテーブルのCRUD操作を実行できます。

```rust
#[derive(Copy, Clone, Default, Debug, DeriveEntity)]
pub struct Entity;

impl EntityName for Entity {
    fn schema_name(&self) -> Option<&str> {
        None
    }

    fn table_name(&self) -> &str {
        "cake"
    }
}
```

### 列

このテーブルのすべての列を表現する列挙型です。

```rust
#[derive(Copy, Clone, Debug, EnumIter, DeriveColumn)]
pub enum Column {
    Id,
    Name,
}

impl ColumnTrait for Column {
    type EntityName = Entity;

    fn def(&self) -> ColumnDef {
        match self {
            Self::Id => ColumnType::Integer.def(),
            Self::Name => ColumnType::String(None).def(),
        }
    }
}
```

すべての列名は、スネークケースで想定されます。
`column_name`属性を指定することで、列の名前を上書きできます。

```rust
pub enum Column {
    Id,      // SQLで"id"にマッピング
    Name,    // SQLで"name"にマッピング
    #[sea_orm(column_name = "create_at")]
    CreateAt // SQLで"create_at"にマッピング
}
```

それぞれの列にデータ型を指定するために、[ColumnTyp](https://docs.rs/sea-orm/*/sea_orm/entity/enum.ColumnType.html)列挙型を使用されます。

#### 追加の属性

- デフォルト値
- ユニーク
- インデックス
- NULL許容

```rust
ColumnType::String(None).def().default_value("Sam").unique().indexed().nullable()
```

#### 選択と保存における列型のキャスト

もし、ある型で列を選択するが、別の方でデータベースに保存する必要がある場合、キャストを実行するために`select_as`と`save_as`属性を記述できます。
典型的なユースケースは、`citext`（テキストの大文字小文字を無視）型の列をRustの`String`で選択して、`citext`としてデータベースに保存することです。
`ColumnTrait`のメソッドを次のように上書きする必要があります。

```rust
use sea_orm::sea_query::{Expr, SimpleExpr, Alias}

impl ColumnTrait for Column {
    // Snipped...

    /// Cast column expression used in select statement.
    fn select_as(&self, expr: Expr) -> SimpleExpr {
        Column::CaseInsensitiveText => expr.cast_as(Alias::new("text")),
        _ => self.select_enum_as(expr),
    }

    /// Cast value of a column into the correct type for database storage.
    fn save_as(&self, val: Expr) -> SimpleExpr {
        Column::CaseInsensitiveText => val.cast_as(Alias::new("citext")),
        _ => self.save_enum_as(val),
    }
}
```

### プライマリーキー

テーブルのプライマリーキーを表現する列挙型です。

複合キーは、複数のバリアントを持つ列挙型によって表現されます。

`ValueType`は、[InsertResult](https://docs.rs/sea-orm/*/sea_orm/struct.InsertResult.html)内の`last_inserted_id`の型を定義します。

`auto_increment`はプライマリーキーが自動で生成された値を持っているかを定義します。

```rust
#[derive(Copy, Clone, Debug, EnumIter, DerivePrimaryKey)]
pub enum PrimaryKey {
    #[sea_orm(column_name = "id")] // デフォルトの列名を上書き
    Id,  // SQLで"id"にマッピング
}

impl PrimaryKeyTrait for PrimaryKey {
    type ValueType = i32;

    fn auto_increment() -> bool {
        true
    }
}
```

次は複合キーの例です。

```rust
pub enum PrimaryKey {
    CakeId,
    FruitId,
}

impl PrimaryKeyTrait for PrimaryKey {
    type ValueType = (i32, i32);

    fn auto_increment() -> bool {
        false
    }
}
```

### モデル

クエリの結果を保存するRustの構造体です。

```rust
#[derive(Clone, Debug, PartialEq, Eq, DeriveModel, DeriveActiveModel)]
pub struct Model {
    pub id: i32,
    pub name: String,
}
```

#### NULL許可属性

テーブルの列がNULLを許容する場合、それを`Option`でラップします。

```rust
pub struct Model {
    pub id: i32,
    pub name: Option<String>,
}
```

### アクティブモデル

`ActiveModel`はそれと対応する`Model`のすべての属性を持ちますが、すべての属性は[ActiveValue](https://docs.rs/sea-orm/*/sea_orm/entity/enum.ActiveValue.html)でラップされています。

```rust
#[derive(Clone, Debug, PartialEq)]
pub struct ActiveModel {
    pub id: ActiveValue<i32>,
    pub name: ActiveValue<Option<String>>,
}
```

### 関連

他のエンティティとの関連を指定します。

```rust
#[derive(Copy, Clone, Debug, EnumIter)]
pub enum Relation {
    Fruit,
}

impl RelationTrait for Relation {
    fn def(&self) -> RelationDef {
        match self {
            Self::Fruit => Entity::has_many(super::fruit::Entity).into(),
        }
    }
}
```

