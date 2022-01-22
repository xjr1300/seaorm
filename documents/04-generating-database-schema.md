# データベーススキーマの生成

## テーブルの作成

データベースにテーブルを作成するために、[TableCreateStatement](https://docs.rs/sea-query/*/sea_query/table/struct.TableCreateStatement.html)を手動で記述する代わりに、[Schema::create_table_from_entity](https://docs.rs/sea-orm/0.5/sea_orm/schema/struct.Schema.html#method.create_table_from_entity)を使用して`Entity`から導き出せる。
このメソッドは、`Entity`に定義されたすべての列と外部キー制約を含むデータベーステーブルを作成することに役立つ。

下で`CakeFillingPrice`エンティティを使用して、生成するSQLステートメントを生成するデモを示す。

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
                    .name("fk-cake_filling_price-cake_filling")
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
`

上記をさらに説明するために、下に文字列でSQLステートメントを示す。

* PostgreSQL

```rust
use sea_orm::{tests_cfg::*, DbBackend, Schema, Statement};

let db_postgres = DbBackend::Postgres;
let schema = Schema::new(db_postgres);

assert_eq!(
    db_postgres.build(&schema.create_table_from_entity(CakeFillingPrice)),
    Statement::from_string(
        db_postgres,
        vec![
            r#"CREATE TABLE "public"."cake_filling_price" ("#,
            r#""cake_id" integer NOT NULL,"#,
            r#""filling_id" integer NOT NULL,"#,
            r#""price" decimal NOT NULL,"#,
            r#"CONSTRAINT "pk-cake_filling_price" PRIMARY KEY ("cake_id", "filling_id"),"#,
            r#"CONSTRAINT "fk-cake_filling_price-cake_filling" FOREIGN KEY ("cake_id", "filling_id") REFERENCES "cake_filling" ("cake_id", "filling_id")"#,
            r#")"#,
        ]
        .join(" ")
    )
);
```

* MySQL

```rust
use sea_orm::{tests_cfg::*, DbBackend, Schema, Statement};

let db_mysql = DbBackend::MySql;
let schema = Schema::new(db_mysql);

assert_eq!(
    db_mysql.build(&schema.create_table_from_entity(CakeFillingPrice)),
    Statement::from_string(
        db_mysql,
        vec![
            "CREATE TABLE `cake_filling_price` (",
            "`cake_id` int NOT NULL,",
            "`filling_id` int NOT NULL,",
            "`price` decimal NOT NULL,",
            "PRIMARY KEY `pk-cake_filling_price` (`cake_id`, `filling_id`),",
            "CONSTRAINT `fk-cake_filling_price-cake_filling` FOREIGN KEY (`cake_id`, `filling_id`) REFERENCES `cake_filling` (`cake_id`, `filling_id`)",
            ")",
        ]
        .join(" ")
    )
);
`

* SQLite

```rust
use sea_orm::{tests_cfg::*, DbBackend, Schema, Statement};

let db_sqlite = DbBackend::Sqlite;
let schema = Schema::new(db_sqlite);

assert_eq!(
    db_sqlite.build(&schema.create_table_from_entity(CakeFillingPrice)),
    Statement::from_string(
        db_sqlite,
        vec![
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

## 列挙体の作成

[Schema](https://docs.rs/sea-orm/0.5/sea_orm/schema/struct.Schema.html)ヘルパー構造体で、列挙体列を持つデータベーステーブルを作成s流ために、SQLステートメントを生成できる。

### 文字列と整数列挙体

これは単にRustの列挙体にマッピングするデータベーステーブル内の通常な文字列型/整数型列であり、丁度、前のセクションのようなテーブルを作成するステートメントを構築する[Schema::create_table_from_entity](https://docs.rs/sea-orm/0.5/sea_orm/schema/struct.Schema.html#method.create_table_from_entity)メソッドを使用できる。

エンティティと列挙体を定義する。

```rust
pub mod active_enum {
    use sea_orm::entity::prelude::*;
    
    #[derive(Clone, Debug, PartialEq, DeriveEntityModel)]
    #[sea_orm(schema_name = "public", table_name = "active_enum")]
    pub struct Model {
        #[sea_orm(primary_key)]
        pub id: i32,
        pub category: Option<Category>,
        pub color: Option<Color>,
    }
    
    #[derive(Debug, Clone, PartialEq, EnumIter, DeriveActiveEnum)]
    #[sea_orm(rs_type = "String", db_type = "String(Some(1))")]
    pub enum Category {
        #[sea_orm(string_value = "B")]
        Big,
        #[sea_orm(string_value = "S")]
        Small,
    }
    
    #[derive(Debug, Clone, PartialEq, EnumIter, DeriveActiveEnum)]
    #[sea_orm(rs_type = "i32", db_type = "Integer")]
    pub enum Color {
        #[sea_orm(num_value = 0)]
        Black,
        #[sea_orm(num_value = 1)]
        White,
    }
    
    #[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
    pub enum Relation {}
    
    impl ActiveModelBehavior for ActiveModel {}
}
```

エンティティから[TableCreateStatement](https://docs.rs/sea-query/*/sea_query/table/struct.TableCreateStatement.html)を生成する。

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

## 本来のデータベースの列挙体

列挙体のサーポートはデータベースによって異なる。
データベースごとに1つずつ本来のデータベース列挙体の作成を説明する。

エンティティと列挙体を定義する。

```rust
pub mod active_enum {
    use sea_orm::entity::prelude::*;
    
    #[derive(Clone, Debug, PartialEq, DeriveEntityModel)]
    #[sea_orm(schema_name = "public", table_name = "active_enum")]
    pub struct Model {
        #[sea_orm(primary_key)]
        pub id: i32,
        pub tea: Option<Tea>,
    }
    
    #[derive(Debug, Clone, PartialEq, EnumIter, DeriveActiveEnum)]
    #[sea_orm(rs_type = "String", db_type = "Enum", enum_name = "tea")]
    pub enum Tea {
        #[sea_orm(string_value = "EverydayTea")]
        EverydayTea,
        #[sea_orm(string_value = "BreakfastTea")]
        BreakfastTea,
    }
    
    #[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
    pub enum Relation {}
    
    impl ActiveModelBehavior for ActiveModel {}
}
```

### PostgreSQL

PostgreSQLにおいて列挙体はカスタム型で定義され、列挙体は[Schema::create_enum_from_entity](https://docs.rs/sea-orm/0.5/sea_orm/schema/struct.Schema.html#method.create_enum_from_entity)メソッドによってエンティティから作成される。

また、[Schema::create_enum_from_active_enum](https://docs.rs/sea-orm/0.5/sea_orm/schema/struct.Schema.html#method.create_enum_from_active_enum)メソッドにより`ActiveEnum`から直接作成することもできる。

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
    vec![Statement::from_string(
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
        vec![
            r#"CREATE TABLE "public"."active_enum" ("#,
            r#""id" serial NOT NULL PRIMARY KEY,"#,
            r#""tea" tea"#,
            r#")"#,
        ]
        .join(" ")
    ),
);
```

### MySQL

MySQLにおいて、列挙体はテーブルの作成で定義されるため、[Schema::create_table_from_entity](https://docs.rs/sea-orm/0.5/sea_orm/schema/struct.Schema.html#method.create_table_from_entity)めそどのみ必要である。

```rust
use sea_orm::{Schema, Statement};

let db_mysql = DbBackend::MySql;
let schema = Schema::new(db_mysql);

assert_eq!(
    db_mysql.build(&schema.create_table_from_entity(active_enum::Entity)),
    Statement::from_string(
        db_mysql,
        vec![
            "CREATE TABLE `active_enum` (",
            "`id` int NOT NULL AUTO_INCREMENT PRIMARY KEY,",
            "`tea` ENUM('EverydayTea', 'BreakfastTea')",
            ")",
        ]
        .join(" ")
    ),
);
```

### SQLite

SQLiteでは列挙体はサポートされておらず、列挙体は`TEXT`型で保存される。

```rust
use sea_orm::{Schema, Statement};

let db_sqlite = DbBackend::Sqlite;
let schema = Schema::new(db_sqlite);

assert_eq!(
    db_sqlite.build(&schema.create_table_from_entity(active_enum::Entity)),
    Statement::from_string(
        db_sqlite,
        vec![
            "CREATE TABLE `active_enum` (",
            "`id` integer NOT NULL PRIMARY KEY AUTOINCREMENT,",
            "`tea` text",
            ")",
        ]
        .join(" ")
    ),
);
```


