# スキーマ文

- [スキーマ文](#スキーマ文)
  - [テーブル作成](#テーブル作成)
    - [PostgreSQL](#postgresql)
    - [MySQL](#mysql)
    - [SQLite](#sqlite)

## テーブル作成

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
