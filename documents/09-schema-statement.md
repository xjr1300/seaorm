# スキーマ文

- [スキーマ文](#スキーマ文)
  - [テーブルの作成](#テーブルの作成)
    - [PostgreSQL](#postgresql)
    - [MySQL](#mysql)
    - [SQLite](#sqlite)
  - [列挙型の作成](#列挙型の作成)
    - [文字列型と整数型の列挙型](#文字列型と整数型の列挙型)
    - [ネイティブなデータベースの列挙型](#ネイティブなデータベースの列挙型)
      - [PostgreSQL](#postgresql-1)
      - [MySQL](#mysql-1)
      - [SQLite](#sqlite-1)

## テーブルの作成

手動で[TableCreateStatement](https://docs.rs/sea-query/*/sea_query/table/struct.TableCreateStatement.html)を記述する代わりに、データベースにテーブルを作成するために、[Schema::create_table_from_entity](https://docs.rs/sea-orm/*/sea_orm/schema/struct.Schema.html#method.create_table_from_entity)を使用して`Entity`からそれを導出できます。

このメソッドは、`Entity`で定義されたすべての列と外部キー制約を含んだ、データベーステーブルを作成する支援をします。

次に、[CakeFillingPrice](https://github.com/SeaQL/sea-orm/blob/master/src/tests_cfg/cake_filling_price.rs)エンティティを使用して、生成されたSQL文をデモします。
[TableCreateStatement](https://docs.rs/sea-query/*/sea_query/table/struct.TableCreateStatement.html)で同じ文を構築することもできます。

バージョン`0.7.0`以降、[Schema::create_table_from_entity](https://docs.rs/sea-orm/*/sea_orm/schema/struct.Schema.html#method.create_table_from_entity)は、もはやインデックスを作成しないことに注意してください。
もし、データベースにインデックスを作成する必要がある場合は、詳細を[ここ](https://www.sea-ql.org/SeaORM/docs/schema-statement/create-index/)で参照してください。

```rust
use sea_orm::{sea_query::*, tests_cfg::*, EntityName, Schema};

let builder = db.get_database_backend();
let schema = Schema::new(builder);

assert_eq!(
    builder.build(&schema.create_table_from_entity(CakeFillingPrice)),
    builder.build(
        &Table::create()
            .table(CakeFillingPrice.table_ref())
            .col(
                ColumnDef::new(cake_filling_price::Column::CakeId)
                    .integer()
                    .not_null(),
            )
            .col(
                ColumnDef::new(cake_filling_price::Column::FillingId)
                    .integer()
                    .not_null(),
            )
            .col(
                ColumnDef::new(cake_filling_price::Column::Price)
                    .decimal()
                    .not_null(),
            )
            .primary_key(
                Index::create()
                    .name("pk-cake_filling_price")
                    .col(cake_filling_price::Column::CakeId)
                    .col(cake_filling_price::Column::FillingId)
                    .primary(),
            )
            .foreign_key(
                ForeignKeyCreateStatement::new()
                    .name("fk-cake_filling_price-cake_id-filling_id")
                    .from_tbl(CakeFillingPrice)
                    .from_col(cake_filling_price::Column::CakeId)
                    .from_col(cake_filling_price::Column::FillingId)
                    .to_tbl(CakeFilling)
                    .to_col(cake_filling::Column::CakeId)
                    .to_col(cake_filling::Column::FillingId),
            )
            .to_owned()
    )
);
```

それを詳しく説明するために、次に文字列でSQL文を示します。

### PostgreSQL

```rust
use sea_orm::{tests_cfg::*, DbBackend, Schema, Statement};

let db_postgres = DbBackend::Postgres;
let schema = Schema::new(db_postgres);

assert_eq!(
    db_postgres.build(&schema.create_table_from_entity(CakeFillingPrice)),
    Statement::from_string(
        db_postgres,
        [
            r#"CREATE TABLE "public"."cake_filling_price" ("#,
            r#""cake_id" integer NOT NULL,"#,
            r#""filling_id" integer NOT NULL,"#,
            r#""price" decimal NOT NULL,"#,
            r#"CONSTRAINT "pk-cake_filling_price" PRIMARY KEY ("cake_id", "filling_id"),"#,
            r#"CONSTRAINT "fk-cake_filling_price-cake_id-filling_id" FOREIGN KEY ("cake_id", "filling_id") REFERENCES "cake_filling" ("cake_id", "filling_id")"#,
            r#")"#,
        ]
        .join(" ")
    )
);
```

### MySQL

```rust
use sea_orm::{tests_cfg::*, DbBackend, Schema, Statement};

let db_mysql = DbBackend::MySql;
let schema = Schema::new(db_mysql);

assert_eq!(
    db_mysql.build(&schema.create_table_from_entity(CakeFillingPrice)),
    Statement::from_string(
        db_mysql,
        [
            "CREATE TABLE `cake_filling_price` (",
            "`cake_id` int NOT NULL,",
            "`filling_id` int NOT NULL,",
            "`price` decimal NOT NULL,",
            "PRIMARY KEY `pk-cake_filling_price` (`cake_id`, `filling_id`),",
            "CONSTRAINT `fk-cake_filling_price-cake_id-filling_id` FOREIGN KEY (`cake_id`, `filling_id`) REFERENCES `cake_filling` (`cake_id`, `filling_id`)",
            ")",
        ]
        .join(" ")
    )
);
```

### SQLite

```rust
use sea_orm::{tests_cfg::*, DbBackend, Schema, Statement};

let db_sqlite = DbBackend::Sqlite;
let schema = Schema::new(db_sqlite);

assert_eq!(
    db_sqlite.build(&schema.create_table_from_entity(CakeFillingPrice)),
    Statement::from_string(
        db_sqlite,
        [
            "CREATE TABLE `cake_filling_price` (",
            "`cake_id` integer NOT NULL,",
            "`filling_id` integer NOT NULL,",
            "`price` real NOT NULL,",
            "CONSTRAINT `pk-cake_filling_price`PRIMARY KEY (`cake_id`, `filling_id`),",
            "FOREIGN KEY (`cake_id`, `filling_id`) REFERENCES `cake_filling` (`cake_id`, `filling_id`)",
            ")",
        ]
        .join(" ")
    )
);
```

## 列挙型の作成

[Schema](https://docs.rs/sea-orm/*/sea_orm/schema/struct.Schema.html)ヘルパー構造体で、列挙型の列を持つデータベーステーブルを作成するSQL文を生成できます。

### 文字列型と整数型の列挙型

これは、Rustの列挙型にマッピングする普通の文字列型／整数型の列です。
エンティティ定義の例を次に示します。

```rust
// active_enum.rs
use sea_orm::entity::prelude::*;

#[derive(Clone, Debug, PartialEq, Eq, DeriveEntityModel)]
#[sea_orm(schema_name = "public", table_name = "active_enum")]
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i32,
    pub category: Option<Category>,
    pub color: Option<Color>,
}

#[derive(Debug, Clone, PartialEq, Eq, EnumIter, DeriveActiveEnum)]
#[sea_orm(rs_type = "String", db_type = "String(Some(1))")]
pub enum Category {
    #[sea_orm(string_value = "B")]
    Big,
    #[sea_orm(string_value = "S")]
    Small,
}

#[derive(Debug, Clone, PartialEq, Eq, EnumIter, DeriveActiveEnum)]
#[sea_orm(rs_type = "i32", db_type = "Integer")]
pub enum Color {
    #[sea_orm(num_value = 0)]
    Black,
    #[sea_orm(num_value = 1)]
    White,
}
```

例えるのであれば、列挙型は単なる普通のデータベース列です。

```rust
use sea_orm::{sea_query, Schema};

let builder = db.get_database_backend();
let schema = Schema::new(builder);

assert_eq!(
    builder.build(&schema.create_table_from_entity(active_enum::Entity)),
    builder.build(
        &sea_query::Table::create()
            .table(active_enum::Entity.table_ref())
            .col(
                sea_query::ColumnDef::new(active_enum::Column::Id)
                    .integer()
                    .not_null()
                    .auto_increment()
                    .primary_key(),
            )
            .col(sea_query::ColumnDef::new(active_enum::Column::Category).string_len(1))
            .col(sea_query::ColumnDef::new(active_enum::Column::Color).integer())
            .to_owned()
    )
);
```

### ネイティブなデータベースの列挙型

列挙型は、異なるデータベースを通じてサポートしています。
それらを1つずつ確認します。

次のエンティティを考えてください。

```rust
// active_enum.rs
use sea_orm::entity::prelude::*;

#[derive(Clone, Debug, PartialEq, Eq, DeriveEntityModel)]
#[sea_orm(schema_name = "public", table_name = "active_enum")]
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i32,
    pub tea: Option<Tea>,
}

#[derive(Debug, Clone, PartialEq, Eq, EnumIter, DeriveActiveEnum)]
#[sea_orm(rs_type = "String", db_type = "Enum", enum_name = "tea")]
pub enum Tea {
    #[sea_orm(string_value = "EverydayTea")]
    EverydayTea,
    #[sea_orm(string_value = "BreakfastTea")]
    BreakfastTea,
}
```

`db_type`と追加の`enum_name`属性に注意してください。

#### PostgreSQL

PostgreSQLの列挙型は[TypeCreateStatement](https://docs.rs/sea-query/*/sea_query/extension/postgres/struct.TypeCreateStatement.html)で定義されており、それは[Schema::create_enum_from_entity](https://docs.rs/sea-orm/*/sea_orm/schema/struct.Schema.html#method.create_enum_from_entity)メソッドで`Entity`から作成されます。

それを[Schema::create_enum_from_active_enum](https://docs.rs/sea-orm/*/sea_orm/schema/struct.Schema.html#method.create_enum_from_active_enum)メソッドで`ActiveEnum`から作成することもできます。

```rust
use sea_orm::{Schema, Statement};

let db_postgres = DbBackend::Postgres;
let schema = Schema::new(db_postgres);

assert_eq!(
    schema
        .create_enum_from_entity(active_enum::Entity)
        .iter()
        .map(|stmt| db_postgres.build(stmt))
        .collect::<Vec<_>>(),
    [Statement::from_string(
        db_postgres,
        r#"CREATE TYPE "tea" AS ENUM ('EverydayTea', 'BreakfastTea')"#.to_owned()
    ),]
);

assert_eq!(
    db_postgres.build(&schema.create_enum_from_active_enum::<Tea>()),
    Statement::from_string(
        db_postgres,
        r#"CREATE TYPE "tea" AS ENUM ('EverydayTea', 'BreakfastTea')"#.to_owned()
    )
);

assert_eq!(
    db_postgres.build(&schema.create_table_from_entity(active_enum::Entity)),
    Statement::from_string(
        db_postgres,
        [
            r#"CREATE TABLE "public"."active_enum" ("#,
            r#""id" serial NOT NULL PRIMARY KEY,"#,
            r#""tea" tea"#,
            r#")"#,
        ]
        .join(" ")
    ),
);
```

#### MySQL

MySQLにおいて、列挙型はテーブル作成で定義されるため、[Schema::create_table_from_entity](https://docs.rs/sea-orm/*/sea_orm/schema/struct.Schema.html#method.create_table_from_entity)をただ1回呼び出す必要があります。

```rust
use sea_orm::{Schema, Statement};

let db_mysql = DbBackend::MySql;
let schema = Schema::new(db_mysql);

assert_eq!(
    db_mysql.build(&schema.create_table_from_entity(active_enum::Entity)),
    Statement::from_string(
        db_mysql,
        [
            "CREATE TABLE `active_enum` (",
            "`id` int NOT NULL AUTO_INCREMENT PRIMARY KEY,",
            "`tea` ENUM('EverydayTea', 'BreakfastTea')",
            ")",
        ]
        .join(" ")
    ),
);
```

#### SQLite

SQLiteで列挙型はサポートされていないため、それは`TEXT`として保存されます。

```rust
use sea_orm::{Schema, Statement};

let db_sqlite = DbBackend::Sqlite;
let schema = Schema::new(db_sqlite);

assert_eq!(
    db_sqlite.build(&schema.create_enum_from_entity(active_enum::Entity)),
    Statement::from_string(
        db_sqlite,
        [
            "CREATE TABLE `active_enum` (",
            "`id` integer NOT NULL PRIMARY KEY AUTOINCREMENT,",
            "`tea` text",
            ")",
        ]
        .join(" ")
    ),
);
```
