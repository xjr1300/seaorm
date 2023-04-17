# マイグレーション

- [マイグレーション](#マイグレーション)
  - [マイグレーションの設定](#マイグレーションの設定)
    - [テーブルのマイグレーション](#テーブルのマイグレーション)
    - [マイグレーションディレクトリの作成](#マイグレーションディレクトリの作成)
    - [ワークスペースの構造](#ワークスペースの構造)
      - [マイグレーションクレート](#マイグレーションクレート)
      - [エンティティクレート](#エンティティクレート)
      - [アプリクレート](#アプリクレート)
  - [マイグレーションの記述](#マイグレーションの記述)
    - [マイグレーションの作成](#マイグレーションの作成)
    - [マイグレーションの定義](#マイグレーションの定義)
      - [SeaQuery](#seaquery)
        - [スキーマ作成メソッド](#スキーマ作成メソッド)
        - [テーブルの作成](#テーブルの作成)
        - [インデックスの作成](#インデックスの作成)
        - [外部キーの作成](#外部キーの作成)
        - [データ型の作成（PostgreSQLのみ）](#データ型の作成postgresqlのみ)
        - [スキーマ変更メソッド](#スキーマ変更メソッド)
        - [テーブル削除](#テーブル削除)
        - [テーブル変更](#テーブル変更)
        - [テーブル名変更](#テーブル名変更)
        - [テーブル行削除（`Truncate`）](#テーブル行削除truncate)
        - [インデックス削除](#インデックス削除)
        - [外部キー削除](#外部キー削除)
        - [データ型変更（PostgreSQLのみ）](#データ型変更postgresqlのみ)
        - [データ型削除（PostgreSQLのみ）](#データ型削除postgresqlのみ)
        - [スキーマ検証メソッド](#スキーマ検証メソッド)
        - [テーブル存在確認](#テーブル存在確認)
        - [列存在確認](#列存在確認)
    - [複数のスキーマの変更を1つのマイグレーションに結合](#複数のスキーマの変更を1つのマイグレーションに結合)
      - [Raw SQL](#raw-sql)
    - [アトミックなマイグレーション](#アトミックなマイグレーション)
    - [スキーマファースト、それともエンティティファースト?](#スキーマファーストそれともエンティティファースト)
  - [マイグレーションの実行](#マイグレーションの実行)
    - [コマンドラインインターフェース（CLI）](#コマンドラインインターフェースcli)
      - [`sea-orm-cli`によって](#sea-orm-cliによって)
      - [SeaSchemaマイグレーターCLIによって](#seaschemaマイグレーターcliによって)
    - [プログラムによるマイグレーション](#プログラムによるマイグレーション)
    - [任意のPostgreSQLスキーマでマイグレーションを適用](#任意のpostgresqlスキーマでマイグレーションを適用)

## マイグレーションの設定

> もし、既にテーブルとスキーマのあるデータベースがある場合、この章をスキップして、[SeaORMのエンティティの生成](https://www.sea-ql.org/SeaORM/docs/generate-entity/sea-orm-cli/)に移動できます。

もし、新しいデータベースから開始している場合、データベーススキーマをバージョン管理することが好ましいです。
SeaORMは、マイグレーションツールと共に提供していて、SeaQueryまたはSQLでマイグレーションを記述できます。

### テーブルのマイグレーション

`seaql_migrations`と名付けられたテーブルは、適用されたマイグレーションを追跡し続けるために、データベース内に作成されます。
`seaql_migrations`は、マイグレーションを実行したとき、自動的に作成されます。

### マイグレーションディレクトリの作成

最初に、`cargo`で`sea-orm-cli`をインストールします。

```bash
cargo install sea-orm-cli
```

次に、`sea-orm-cli migrate init`を実行することにより、マイグレーションディレクトリを準備します。

```bash

# `./migration`内にマイグレーションディレクトリを準備
$ sea-orm-cli migrate init
Initializing migration directory...
Creating file `./migration/src/lib.rs`
Creating file `./migration/src/m20220101_000001_create_table.rs`
Creating file `./migration/src/main.rs`
Creating file `./migration/Cargo.toml`
Creating file `./migration/README.md`
Done!

# もし、他の場所にマイグレーションディレクトリを準備したい場合
$ sea-orm-cli migrate init -d ./other/migration/dir
```

下のような構造のマイグレーションディレクトリがあるはずです。

```text
migration
├── Cargo.toml
├── README.md
└── src
    ├── lib.rs                              # 統合するためのマイグレーターAPI
    ├── m20220101_000001_create_table.rs    # マイグレーションファイルのサンプル
    └── main.rs                             # 手動で実行するためのマイグレーター
```

### ワークスペースの構造

アプリクレートとマイグレーションクレート間で、SeaORMエンティティを共有するために、次のとおりcargoワークスペースを構造化することを推奨します。
説明のために[統合例](https://github.com/SeaQL/sea-orm/tree/master/examples)を確認してください。

#### マイグレーションクレート

[sea-orm-migration](https://crates.io/crates/sea-orm-migration)と[async-std](https://crates.io/crates/async-std)クレートをインポートします。

```toml
# migration/Cargo.toml
[dependencies]
async-std = { version = "^1", features = ["attributes", "tokio1"] }

[dependencies.sea-orm-migration]
version = "^0"
features = [
  # CLIによってマイグレーションを実行するために、少なくとも1つの`ASYNC_RUNTIME`と`DATABASE_DRIVER`フィーチャを有効にしてください。
  # https://www.sea-ql.org/SeaORM/docs/install-and-config/database-and-async-runtime でサポートされたフィーチャのリストを参照できます。
  # 例えば
  # "runtime-tokio-rustls",  # `ASYNC_RUNTIME`フィーチャ
  # "sqlx-postgres",         # `DATABASE_DRIVER`フィーチャ
]
```

マイグレーションを記述します。
詳細は次の節で説明します。

```rust
// migration/src/m20220120_000001_create_post_table.rs
use sea_orm_migration::prelude::*;

#[derive(DeriveMigrationName)]
pub struct Migration;

#[async_trait]
impl MigrationTrait for Migration {
    async fn up(&self, manager: &SchemaManager) -> Result<(), DbErr> {
        // Replace the sample below with your own migration scripts
        todo!();
    }

    async fn down(&self, manager: &SchemaManager) -> Result<(), DbErr> {
        // Replace the sample below with your own migration scripts
        todo!();
    }
}
```

#### エンティティクレート

ルートワークスペース内にエンティティクレートを作成します。

> 定義されたSeaORMエンティティがありませんか?
>
> エンティティファイルなしでエンティティクレートを作成できます。
> そして、マイグレーションを記述して、データベース内にテーブルを作成するためにそれを実行します。
> 最後に、`sea-orm-cli`でSeaORMエンティティを生成して、`entity/src/entities`フォルダにエンティティファイルを出力します。
>
> エンティティファイルを生成した後で、`entity/src/lib.rs`に次の行を追加することで、生成したエンティティファイルを再エクスポートできます。
>
> ```rust
> mod entities;
> pub use entities::*;
> ```

```text
entity
├── Cargo.toml      # SeaORMへの依存を含める
└── src
    ├── lib.rs      # SeaORMとエンティティを再エクスポート
    └── post.rs     # `post`エンティティを定義
```

SeaORMへの依存を記述します。

```toml
# entity/Cargo.toml
[dependencies]
sea-orm = { version = "^0" }
```

#### アプリクレート

これは、アプリケーションロジックが実行される場所です。

アプリ、エンティティとマイグレーションクレートを含んだワークスペースを作成して、アプリクレートにエンティティクレートをインポートします。

もし、アプリの部分としてマイグレーションユーティリティをまとめたい場合は、マイグレーションクレートもインポートする必要があります。

```toml
# ./Cargo.toml
[workspace]
members = [".", "entity", "migration"]

[dependencies]
entity = { path = "entity" }
migration = { path = "migration" } # depends on your needs

[dependencies.sea-orm]
version = "^0"
features = [ ... ]
```

アプリの開始でマイグレーションを実行できます。

```rust
// src/main.rs
use migration::{Migrator, MigratorTrait};

let connection = sea_orm::Database::connect(&database_url).await?;
Migrator::up(&connection, None).await?;
```

## マイグレーションの記述

それぞれのマイグレーションは、`up`と`down`の2つのメソッドを含んでいます。
`up`メソッドは、新しいテーブル、列またはインデックスを追加するようなデータベーススキーマを変更するために使用され、一方`down`メソッドは、`up`メソッドによって実行されたアクションを元に戻します。

### マイグレーションの作成

`sea-orm-cli migrate generate`コマンドを実行することにより、新しいマイグレーションファイルを作成します。

```bash
sea-orm-cli migrate generate NAME_OF_MIGRATION [--local-time]

# 例えば、下に表示された`migration/src/m20220101_000001_create_table.rs`を生成するためには次を実行
sea-orm-cli migrate generate create_table
```

下のテンプレートを使用してマイグレーションファイルを作成することもできます。
ファイルの名前は`mYYYYMMDD_HHMMSS_migration_name.rs`の形式に従います。

```rust
// migration/src/m20220101_000001_create_table.rs
use sea_orm_migration::prelude::*;

#[derive(DeriveMigrationName)]
pub struct Migration;

#[async_trait]
impl MigrationTrait for Migration {
    async fn up(&self, manager: &SchemaManager) -> Result<(), DbErr> {
        manager
            .create_table( ... )
            .await
    }

    async fn down(&self, manager: &SchemaManager) -> Result<(), DbErr> {
        manager
            .drop_table( ... )
            .await
    }
}
```

さらに、[MigratorTrait::migrations](https://docs.rs/sea-orm-migration/*/sea_orm_migration/migrator/trait.MigratorTrait.html#tymethod.migrations)メソッドの中に新しいマイグレーションを含める必要があります。
マイグレーションが時間の順番でソートされていなければならないことに注意してください。

```rust
// migration/src/lib.rs
pub use sea_orm_migration::*;

mod m20220101_000001_create_table;

pub struct Migrator;

#[async_trait]
impl MigratorTrait for Migrator {
    fn migrations() -> Vec<Box<dyn MigrationTrait>> {
        vec![
            Box::new(m20220101_000001_create_table::Migration),
        ]
    }
}
```

### マイグレーションの定義

APIリファレンスの[SchemaManager](https://docs.rs/sea-orm-migration/*/sea_orm_migration/manager/struct.SchemaManager.html)を参照してください。

> `SchemaManger`構造体は、`create_table`、`create_index`メソッドなど、SQLのDDL文を実行するために必要なメソッドが定義されています。

#### SeaQuery

SeaQueryのDDL文のクイックツアーを受けるために[ここ](https://github.com/SeaQL/sea-query#table-create)をクリックしてください。

マイグレーション内で使用される識別子を定義するために、[sea_query::Iden](https://github.com/SeaQL/sea-query#iden)が必要になるかもしれません。

```rust
#[derive(Iden)]
enum Post {
    Table,
    Id,
    Title,
    #[iden = "text"] // 識別子の名前を変更
    Text,
    Category,
}

assert_eq!(Post::Table.to_string(), "post");
assert_eq!(Post::Id.to_string(), "id");
assert_eq!(Post::Title.to_string(), "title");
assert_eq!(Post::Text.to_string(), "text");
```

##### スキーマ作成メソッド

##### テーブルの作成

```rust
use sea_orm::{EnumIter, Iterable};

#[derive(Iden)]
enum Post {
    Table,
    Id,
    Title,
    #[iden = "text"] // 識別子の名前を変更
    Text,
    Category,
}

#[derive(Iden, EnumIter)]
pub enum Category {
    Table,
    #[iden = "Feed"]
    Feed,
    #[iden = "Story"]
    Story,
}

manager
    .create_table(
        Table::create()
            .table(Post::Table)
            .if_not_exists()
            .col(
                ColumnDef::new(Post::Id)
                    .integer()
                    .not_null()
                    .auto_increment()
                    .primary_key(),
            )
            .col(ColumnDef::new(Post::Title).string().not_null())
            .col(ColumnDef::new(Post::Text).string().not_null())
            .col(
                ColumnDef::new(Column::Category)
                    .enumeration(Category, [Category::Feed, Category::Story]),
                    // または、下のように記述します。
                    // 機能するために必要なことを覚えておいてください。
                    // 1. `EnumIter`を導出する必要があります。
                    // 2. スコープ内に`Iterable`をインポートして、
                    // 3. `Category::Table`が最初のバリアントであることを確認してください。
                    .enumeration(Category, Category::iter().skip(1)),
            )
            .to_owned(),
    )
    .await
```

##### インデックスの作成

```rust
manager.create_index(sea_query::Index::create()..)
```

##### 外部キーの作成

```rust
manager.create_foreign_key(sea_query::ForeignKey::create()..)
```

##### データ型の作成（PostgreSQLのみ）

```rust
use sea_orm::{EnumIter, Iterable};

#[derive(Iden, EnumIter)]
pub enum Category {
    Table,
    #[iden = "Feed"]
    Feed,
    #[iden = "Story"]
    Story,
}

manager
    .create_type(
        Type::create()
            .as_enum(Category::Table)
            .values([Category::Feed, Category::Story])
            // または、下のように記述します。
            // 機能するために必要なことを覚えておいてください。
            // 1. `EnumIter`を導出する必要があります。
            // 2. スコープ内に`Iterable`をインポートして、
            // 3. `Category::Table`が最初のバリアントであることを確認してください。
            .values(Category::iter().skip(1))
            .to_owned(),
    )
    .await?;
```

##### スキーマ変更メソッド

##### テーブル削除

```rust
use entity::post;

manager.drop_table(sea_query::Table::drop()..)
```

##### テーブル変更

```rust
manager.alter_table(sea_query::Table::alter()..)
```

##### テーブル名変更

```rust
manager.rename_table(sea_query::Table::rename()..)
```

##### テーブル行削除（`Truncate`）

```rust
manager.truncate_table(sea_query::Table::truncate()..)
```

##### インデックス削除

```rust
manager.drop_index(sea_query::Index::drop()..)
```

##### 外部キー削除

```rust
manager.drop_foreign_key(sea_query::ForeignKey::drop()..)
```

##### データ型変更（PostgreSQLのみ）

```rust
manager.alter_type(sea_query::Type::alter()..)
```

##### データ型削除（PostgreSQLのみ）

```rust
manager.drop_type(sea_query::Type::drop()..)
```

##### スキーマ検証メソッド

##### テーブル存在確認

```rust
manager.has_table(table_name)
```

##### 列存在確認

```rust
manager.has_column(table_name, column_name)
```

### 複数のスキーマの変更を1つのマイグレーションに結合

`up`と`down`の両方のマイグレーション関数で、複数の変更を結合できます。
ここに完全な例を示します。

```rust
async fn up(&self, manager: &SchemaManager) -> Result<(), DbErr> {
    manager
        .create_table(
            sea_query::Table::create()
                .table(Post::Table)
                .if_not_exists()
                .col(
                    ColumnDef::new(Post::Id)
                        .integer()
                        .not_null()
                        .auto_increment()
                        .primary_key()
                )
                .col(ColumnDef::new(Post::Title).string().not_null())
                .col(ColumnDef::new(Post::Text).string().not_null())
                .to_owned()
        )
        .await?
    manager
        .create_index(
            Index::create()
                .if_not_exists()
                .name("idx-post_title")
                .table(Post::Table)
                .col(Post::Title)
                .to_owned(),
        )
        .await?;

    Ok(()) // すべて成功!
}
```

そして、これが対応する`down`関数です。

```rust
async fn down(&self, manager: &SchemaManager) -> Result<(), DbErr> {
    manager.drop_index(Index::drop().name("idx-post-title").to_owned())
    .await?;
    manager.drop_table(Table::drop().table(Post::Table).to_owned())
    .await?;
    Ok(()) // すべて成功!
}
```

#### Raw SQL

SQL文でマイグレーションファイルを記述できますが、SeaQueryが提供する複数バックエンドの互換性を失います。

```rust
// migration/src/m20220101_000001_create_table.rs
use sea_orm::Statement;
use sea_orm_migration::prelude::*;

#[derive(DeriveMigrationName)]
pub struct Migration;

#[async_trait]
impl MigrationTrait for Migration {
    async fn up(&self, manager: &SchemaManager) -> Result<(), DbErr> {
        let db = manager.get_connection();
        // SQL文がバインディングする値を持っていない場合は`execute_unprepared`を使用
        db.execute_unprepared(
            "CREATE TABLE `cake` (
                `id` int NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `name` varchar(255) NOT NULL
            )"
        )
        .await?;
        // SQLがバインディングする値を含んでいる場合は`文`を構築
        let stmt = Statement::from_sql_and_values(
            manager.get_database_backend(),
            r#"INSERT INTO `cake` (`name`) VALUES (?)"#,
            ["Cheese Cake".into()]
        );
        db.execute(stmt).await?;

        Ok(())
    }

    async fn down(&self, manager: &SchemaManager) -> Result<(), DbErr> {
        manager
            .get_connection()
            .execute_unprepared("DROP TABLE `cake`")
            .await?;

        Ok(())
    }
}
```

### アトミックなマイグレーション

Postgresにおいて、マイグレーションはアトミックで実行され、それはマイグレーションファイルがトランザクション内で実行されることを意味しています。
もし、マイグレーションが失敗した場合、データベースへの変更はロールバックされます。
しかし、アトミックなマイグレーションは、MySQLやSQLiteでサポートされていません。

新しく作成されたテーブルに[サンプルデータを与える](https://www.sea-ql.org/SeaORM/docs/migration/seeding-data/#seeding-data-transactionally)のような操作を実行するために、それぞれのマイグレーション内でトランザクションを開始できます。

### スキーマファースト、それともエンティティファースト?

大局からみれば、スキーマファーストを推奨します。最初にマイグレーションを記述して、実際のデータベースからエンティティを生成します。

ときには、手動で記述されたいくつかのエンティティファイルでデータベースを開始するために、[create_*_from_entity](https://www.sea-ql.org/SeaORM/docs/schema-statement/create-table/)メソッドを使用する必要があるかもしれません。

もし、エンティティのスキーマを決して変更しないことを意図しているか、オリジナルのエンティティを維持して、マイグレーションファイル内にコピーを埋め込むのであれば、それは完全に機能します。

## マイグレーションの実行

マイグレーションを定義した後、ターミナルまたはアプリケーションの開始で、マイグレーションを適用または元に戻せます。

### コマンドラインインターフェース（CLI）

マイグレーションは、ターミナルで手動で実行できます。
`DATABASE_URL`は、環境変数に設定されていなければならず、[ここ](https://www.sea-ql.org/SeaORM/docs/generate-entity/sea-orm-cli/#configure-environment)の命令に従って設定してください。

サポートされているコマンドは次のとおりです。

- `init`: マイグレーションディレクトリを初期化
- `generate`: 新しいマイグレーションファイルを生成
- `up`: 保留されているマイグレーションを適用
- `up -n 10`: 保留されている10個のマイグレーションを適用
- `down`: 最後に適用されたマイグレーションをロールバック
- `down -n 10`: 適用された最後の10個のマイグレーションをロールバック
- `status`: すべてのマイグレーションの状態を確認
- `fresh`: データベースからすべてのテーブルを削除した後、すべてのマイグレーションを再適用
- `refresh`: 適用されたすべてのマイグレーションをロールバックした後、すべてのマイグレーションを再適用
- `reset`: 適用されたすべてのマイグレーションをロールバック

#### `sea-orm-cli`によって

`sea-rom-cli`は、内部で`cargo run --manifest-path ./migration/Cargo.toml -- COMMAND`を実行します。

```bash
sea-orm-cli migrate COMMAND
```

マニフェストのパスをカスタマイズできます。

```bash
sea-orm-cli migrate COMMAND -d ./other/migration/dir
```

#### SeaSchemaマイグレーターCLIによって

`migration/main.rs`に定義されたマイグレーターCLIを実行します。

```bash
cd migration
cargo run -- COMMAND
```

### プログラムによるマイグレーション

[MigratorTrait](https://docs.rs/sea-orm-migration/*/sea_orm_migration/migrator/trait.MigratorTrait.html)を実装する`Migrator`でアプリケーションの開始時にマイグレーションを実行できます。

```rust
// src/main.rs
use migration::{Migrator, MigratorTrait};

/// 保留されているすべてのマイグレーションを適用
Migrator::up(db, None).await?;

/// 保留されている10個のマイグレーションを適用
Migrator::up(db, Some(10)).await?;

/// 最後に適用されたマイグレーションをロールバック
Migrator::down(db, None).await?;

/// 適用された最後の10個のマイグレーションをロールバック
Migrator::down(db, Some(10)).await?;

/// すべてのマイグレーションの状態を確認
Migrator::status(db).await?;

/// データベースからすべてのテーブルを削除した後、すべてのマイグレーションを再適用
Migrator::fresh(db).await?;

/// すべてのマイグレーションをロールバックした後、すべてのマイグレーションを再適用
Migrator::refresh(db).await?;

/// 適用されたすべてのマイグレーションをロールバック
Migrator::reset(db).await?;
```

### 任意のPostgreSQLスキーマでマイグレーションを適用

デフォルトでマイグレーションは`public`スキーマで実行されますが、CLIまたはプログラムでマイグレーションを実行するときに、それを上書きできます。

CLIでは、`-s`または`--database_schema`オプションで目的のスキーマを指定できます。

- sea-orm-cliによって: `sea-orm-cli migrate -u postgres://root:root@localhost/database -s my_schema`
- SeaORMマイグレーターによって: `cargo run -- -u postgres://root:root@localhost/database -s my_schema`

プログラムでも目的のスキーマにマイグレーションを実行できます。

```rust
let connect_options = ConnectOptions::new("postgres://root:root@localhost/database".into())
    .set_schema_search_path("my_schema".into()) // デフォルトのスキーマを上書き
    .to_owned();

let db = Database::connect(connect_options).await?

migration::Migrator::up(&db, None).await?;
```
