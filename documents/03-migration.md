# マイグレーション

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
    ├── lib.rs                              # 統合するためのマイグレータAPI
    ├── m20220101_000001_create_table.rs    # マイグレーションファイルのサンプル
    └── main.rs                             # 手動で実行するためのマイグレータ
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
  # CLIを介してマイグレーションを実行するために、少なくとも1つの`ASYNC_RUNTIME`と`DATABASE_DRIVER`フィーチャを有効にしてください。
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
