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
      - [列の型](#列の型)
      - [追加プロパティ](#追加プロパティ)
      - [無視属性](#無視属性)
    - [プライマリーキー](#プライマリーキー)
      - [自動インクリメント](#自動インクリメント)
      - [複合キー](#複合キー)
    - [関連](#関連)
  - [エンティティ構造体の拡張](#エンティティ構造体の拡張)
    - [エンティティ](#エンティティ-1)
    - [列](#列-1)
      - [追加のプロパティ](#追加のプロパティ)
    - [プライマリーキー](#プライマリーキー-1)
    - [モデル](#モデル)
      - [NULL可能属性](#null可能属性)
    - [アクティブモデル](#アクティブモデル)
      - [アクティブモデルの振る舞い](#アクティブモデルの振る舞い)
    - [関連](#関連-1)
    - [関係](#関係)
  - [列挙型](#列挙型)
    - [文字列](#文字列)
    - [整数](#整数)
      - [データベースの列挙型](#データベースの列挙型)
    - [実装](#実装)
      - [Deriveの実装](#deriveの実装)
      - [手動実装](#手動実装)
    - [モデルにおけるActiveEnumの使用](#モデルにおけるactiveenumの使用)

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

データベース内のすべてのテーブルを探して、それぞれ対応する`SeaORM`エンティティファイルを生成する。

サポートしているデータベースを以下に示す。

* `MySQL`
* `PostgreSQL`
* `SQLite`

コマンドラインオプションを以下に示す。

* `-u / --database-url`: データベースURL(デフォルトは環境変数に設定された`DATABASE_URL`)
* `-s / --database-schema`: データベーススキーマ(デフォルトは環境変数に設定された`DATABASE_SCHEMA`)
  * `MySQL`と`SQLite`の場合、この引数は無視される。
  * `PostgreSQL`の場合、この引数はデフォルト値`public`を持つオプションである。
* `-o / --output-dir`: エンティティファイルを出力するディレクトリ(デフォルトはカレントディレクトリ)
* `-v / --verbose`: デバッグメッセージを表示
* `--include-hidden-tables`: 隠しテーブルからエンティティファイルを生成(アンダースコアで名前が始まるテーブルは隠されていて、デフォルトでは無視する)
* `--compact-format`: [コンパクトフォーマット](https://www.sea-ql.org/SeaORM/docs/generate-entity/entity-structure)でエンティティファイルを生成(デフォルトは有効)
* `--expand-format`: [展開されたフォーマット](https://www.sea-ql.org/SeaORM/docs/generate-entity/expanded-entity-structure)でエンティティファイルを生成
* `--with-serde`: 自動的にエンティティを`serde`の`Serialize`/`Deserialize`トレイトから導出(デフォルトは`none`)
  * `none`
  * `serialize`
  * `deserialize`
  * `both`

```bash
# `bakery`データベースのエンティティファイルを生成して、`src/entity`ディレクトリに出力する。
sea-orm-cli generate entity \
    -u sql://sea:sea@localhost/bakery \
    -o src/entity
```

## エンティティ構造体

単純な[Cake](https://github.com/SeaQL/sea-orm/blob/master/src/tests_cfg/cake.rs)エンティティを見なさい。

```rust
use sea_orm::entity::prelude::*;

#[derive(Clone, Debug, PartialEq, DeriveEntityModel)]
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

impl ActiveModelBehavior for ActiveModel {}
```

### エンティティ

`DeriveEntityModel`マクロは、`Model`、`Column`と`PrimaryKey`を関連付けて`Entity`を定義する面倒な作業をすべて実施する。

#### テーブル名

`table_name`属性は、データベース内の対応するテーブルを指定する。
オプションで、データベーススキーマまたはデータベース名を`schema_name`で指定できる。

```rust
#[sea_orm(table_name = "cake", schema_name = "public")]
pub struct Cake { ... }
```

### 列

#### 列の型

列の型は、以下のマッピングで自動的に導出される。

| `Rust`の型                                             | データベースの型(`ColumnType`) |
| ------------------------------------------------------ | ------------------------------ |
| `char`                                                 | `Char`                         |
| `String`                                               | `String`                       |
| `u8`, `i8`                                             | `TinyInteger`                  |
| `u16`, `i16`                                           | `SmallInteger`                 |
| `u32`, `i32`                                           | `Integer`                      |
| `u64`, `i64`                                           | `BigInteger`                   |
| `f32`                                                  | `Float`                        |
| `f64`                                                  | `Double`                       |
| `bool`                                                 | `Boolean`                      |
| `NaiveDate`                                            | `Date`                         |
| `NaiveTime`                                            | `Time`                         |
| `DateTime (chrono::NaiveDateTime)`                     | `DateTime`                     |
| `DateTimeWithTimeZone (chrono::DateTime<FixedOffset>)` | `TimestampWithTimeZone`        |
| `Uuid (uuid::Uuid)`                                    | `Uuid`                         |
| `Json (serde_json::Value)`                             | `Json`                         |
| `Decimal(rust_decimal::Decimal)`                       | `Decimal`                      |
| `Vec<u8>`                                              | `Binary`                       |

`column_type`属性によって、`Rust`の型と`ColumnType`のデフォルトマッピングをオーバーライドできる。

```rust
#[sea_orm(column_type = "Text")]
pub name: String,
```

#### 追加プロパティ

`unique`、`indexed`及び`nullable`などの追加のプロパティを列に追加できる。

オプションの属性のためにカスタム`column_type`を指定する場合、`nullable`も指定する必要がある。

```rust
#[sea_orm(column_type = "Text", unique, indexed, nullable)]
pub name: Option<String>,
```

#### 無視属性

データベースにマッピングしたくないような、特定のモデルの属性を無視したい場合、`ignore`アノテーションを使用できる。

```rust
#[sea_orm(ignore)]
pub ignore_me: String,
```

### プライマリーキー

プライマリーキーとして指定したい列に`primary_key`属性を使用できる。

```rust
#[sea_orm(primary_key)]
pub id: i32,
```

#### 自動インクリメント

デフォルトで、`auto_increment`属性が`primary_key`列に暗黙的に追加される。
`auto_increment`属性に`false`を指定することで、オーバーライドできる。

```rust
#[sea_orm(primary_key, auto_increment = false)]
pub id: i32,
```

#### 複合キー

 一般的に連関テーブルの場合、2つの列の組がプライマリキーとして使用される。
 単純に、複合プライマリキーとして定義したい複数の列に注釈できる。
 デフォルトで、複合キーの`auto_increment`は`false`である。

 ```rust
 pub struct Model {
     #[sea_orm(primary_key)]
     pub cake_id: i32,
     #[sea_orm(primary_key)]
     pub fruit_id: i32,
 }
 ```

### 関連

`DeriveRelation`マクロは、`RelationTrait`の実装の単純なラッパーである。

```rust
#[derive(DeriveRelation)]
pub enum Relation {
    #[sea_orm(has_many = "super::fruit::Entity")]
    Fruit,
}
```

上記は以下に展開される。

```rust
impl RelationTrait for Relation {
    fn def(&self) -> RelationDef {
        match self {
            Self::Fruit => Entity::has_many(super::fruit::Entity).into(),
        }
    }
}
```

次のセクションで、よりリレーションについて学ぶ。

## エンティティ構造体の拡張

`SeaORM`は動的で、実行時に柔軟に構成できることを示す。
`DeriveEntityModel`が何に拡張されるか知りたい場合は読み進めること。
そうでなければ、現在のところスキップできる。

エンティティフォーマットの拡張は、`--expanded-format`オプションで`sea-orm-cli`によって生成される。

### エンティティ

[EntityTrait](https://docs.rs/sea-orm/0.5/sea_orm/entity/trait.EntityTrait.html)の実装によって、与えられたテーブルの`CRUD`操作ができる。

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

`enum`は、このテーブル内のすべての列を表現する。

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

すべての列名はスネークケースで類推される。

```rust
pub enum Column {
    Id,     // SQLで"id"にマッピング
    Name,   // SQLで"name"にマッピング
    CreateAt,   // SQLで"create_at"にマッピング
}
```

それぞれの列にデータ型を指定するために、[ColumnType](https://docs.rs/sea-orm/0.5/sea_orm/entity/enum.ColumnType.html)列挙型を使用できる。

#### 追加のプロパティ

* Unique
* Indexed
* Nullable

```rust
ColumnType::String(None).def().unique().indexed().nullable()
```

### プライマリーキー

テーブルのプライマリキーを表現する列挙型である。
複合キーは複数のバリアントの列挙型で表現される。

`ValueType`は[InsertResult](https://docs.rs/sea-orm/0.5/sea_orm/struct.InsertResult.html)の`last_insert_id`の型を定義する。

`auto_increment`は、プライマリーキーが自動的に生成された値をもつかを指定する。

```rust
#[derive(Copy, Clone, Debug, EnumIter, DerivePrimaryKey)]
pub enum PrimaryKey {
    Id,
}

impl PrimaryKeyTrait for PrimaryKey {
    type ValueType = i32;

    fn auto_increment() -> bool {
        true
    }
}
```

複合キーの例を以下に示す。

```rust
pub enum PrimaryKey {
    CakeId,
    FruitId,
}

impl PrimaryKeyTrait for PrimaryKey {
    type ValueType = (i32, i32,);

    fn auto_increment() -> bool {
        false
    }
}
```

### モデル

クエリの結果を保存する`Rust`構造体である。

```rust
#[derive(Clone, Debug, PartialEq, DeriveModel, DeriveActiveModel)]
pub struct Model {
    pub id: i32,
    pub name: String,
}
```

#### NULL可能属性

テーブルの列がNullを許容する場合、その列を`Option`でラップする。

```rust
pub struct Model {
    pub id: i32,
    pub name: Option<string>,
}
```

### アクティブモデル

`ActiveModel`は`Model`に対応する属性をすべて持つが、すべての属性は[ActiveValue](https://docs.rs/sea-orm/0.5/sea_orm/entity/struct.ActiveValue.html)でラップされる。

```rust
#[derive(Clone, Debug, PartialEq)]
pub struct ActiveModel {
    pub id: ActiveValue<i32>,
    pub name: ActiveValue<Option<String>>,
}
```

#### アクティブモデルの振る舞い

`ActiveModel`でトリガーされた様々なアクションのハンドラである。
例えば、カスタム検証ロジックを実行して、モデルがデータベースに保存されないようにすることができる。
もしトランザクションの内部であれば、検証の終了後、アクションを中断できる。

```rust
impl ActiveModelBehavior for ActiveModel {
    /// デフォルト値で新しいActiveModelを作成する。
    // `Default::default()`も使用できる。
    fn new() -> Self {
        Self {
            uuid: Set(Uuid::new_v4()),
            ..ActiveModelTrait::default(),
        }
    }

    /// 挿入/更新前にトリガーされる。
    fn before_save(self, insert: bool) -> Result<Self, DbErr> {
        if self.price.as_ref() <= &0.0 {
            Err(DbErr::Custom(format!(
                "[before_save] Invalid price, insert: {}",
                insert
            )))
        } else {
            Ok(self)
        }
    }

    /// 挿入/更新後にトリガーされる。
    fn after_save(model: Model, insert: bool) -> Result<Model, DbErr> {
        Ok(model)
    }

    /// 削除前にトリガーされる。
    fn before_delete(self) -> Result<Self, DbError> {
        Ok(self)
    }

    /// 削除後にトリガーされる。
    fn after_delete(self) -> Result<Self, DbError> {
        Ok(self)
    }
}
```

もし、カスタマイズが必要ないのであれば、単に以下のように記述できる。

```rust
impl ActiveModelBehavior for ActiveMode {}
```

### 関連

他のエンティティとの関連を指定する。

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

### 関係

関連するエンティティを同時にクエリする際に役立つトレイト境界を定義して、特に多対多の関連で役に立つ。

```rust
impl Related<super::fruit::Entity> for Entity {
    fn to() -> RelationDef {
        Relation::Fruit.def()
    }
}

impl Related<super::filling::Entity> for Entity {
    fn to() -> RelationDef {
        super::cake_filling::Relation::Filling.def()
    }

    fn via() -> Option<RelationDef> {
        Some(super::cake_filling::Relation::Cake.def().rev())
    }
}
```

## 列挙型

データベースの文字列、整数または列挙型にマッピングされた値があるモデルで、`Rust`の列挙型を使用できる。

### 文字列

```rust
#[derive(EnumIter, DeriveActiveEnum)]
#[sea_orm(rs_type = "String", db_type = "String(Some(1))")]
pub enum Category {
    #[sea_orm(string_value = "B")]
    Big,
    #[sea_orm(string_value = "S")]
    Small,
}
```

### 整数

```rust
#[derive(EnumIter, DeriveActionEnum)]
#[sea_orm(rs_type = "i32", db_type = "Integer")]
pub enum Color {
    #[sea_orm(num_value = 0)]
    Black,
    #[sea_orm(num_value = 1)]
    White,
}
```

#### データベースの列挙型

```rust
#[derive(EnumIter, DeriveActiveEnum)]
#[sea_orm(rs_type = "String", db_type = "Enum", enum_name="tea")]
pub enum Tea {
    #[sea_orm(string_value = "EverydayTea")]
    EverydayTea,
    #[sea_orm(string_value = "BreakfastTea")]
    BreakfastTea,
}
```

### 実装

手動または[DeriveActiveEnum](https://docs.rs/sea-orm/0.5/sea_orm/derive.DeriveActiveEnum.html)導出マクロを使用して、[ActiveEnum](https://docs.rs/sea-orm/0.5/sea_orm/entity/trait.ActiveEnum.html)を実装できる。

#### Deriveの実装

マクロの属性のすべての仕様を確認するために、[DeriveActiveEnum](https://docs.rs/sea-orm/0.5/sea_orm/derive.DeriveActiveEnum.html)を参照しなさい。

```rust
use sea_orm::entity::prelude::*;

// 導出マクロを使用する。
#[derive(Debug, PartialEq, EnumIter, DeriveActiveEnum)]
#[sea_orm(rs_type = "String", db_type = "String(Some(1))", enum_name = "category")]
pub enum Category {
    #[sea_orm(string_value = "B")]
    Big,
    #[sea_orm(string_value = "S")]
    Small,
}
```

#### 手動実装

```rust
use sea_orm::entity::prelude::*;

// 手動で実装する。
#[derive(Debug, PartialEq, EnumIter)]
pub enum Category {
    Big,
    Small,
}

impl ActiveEnum for Category {
    // マクロの`rs_type`をここに貼り付けられる。
    type Value = String;

    // デフォルトで、enum_nameが明示的に提供されていない場合、キャメルケースのRust列挙体の名前になる。
    fn name() -> String {
        "category".to_owned()
    }

    // Rust列挙体のバリアントと`num_value`または`string_value`と対応させる。
    fn to_value(&self) -> Self::Value {
        match self {
            Self::Big => "B",
            Self::Small => "S",
        }
        .to_owned()
    }

    // `num_value`または`string_value`とRust列挙体のバリアントを対応させる。
    fn try_from_value(v: &Self::Value) -> Result<Self, DbErr> {
        match v.as_ref() {
            "B" => Ok(Self::Big),
            "S" => Ok(Self::Small),
            _ => Err(DbErr::Type(format!("unexpected value for Category enum: {}", v))),
        }
    }

    // マクロの`db_type`属性がここに貼り付けされる。
    fn db_type() -> ColumnDef {
        ColumnDef::String(Some(1)).def()
    }
}
```

### モデルにおけるActiveEnumの使用

```rust
use sea_orm::entity::prelude::*;

#[derive(Debug, Clone, PartialEq, EnumIter, DeriveActiveEnum)]
pub enum Category {
    #[sea_orm(string_value = "B")]
    Big,
    #[sea_orm(string_value = "S")]
    Small,
}

#[derive(Clone, Debug, PartialEq, DeriveEntityModel)]
#[sea_orm(table_name = "active_enum")]
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i32,
    pub category: Category,
    pub category_opt: Option<Category>,
}

#[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
pub enum Relation {}

impl ActiveModelBehavior for ActiveModel {}
```
