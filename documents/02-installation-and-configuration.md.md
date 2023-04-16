# インストールと設定

## データベースと非同期ランタイム

まず最初に、[GitHubリポジトリ](https://github.com/SeaQL/sea-orm)にスターを付けてください!
あなたの支援がSeaORMの維持を継続するために重要です。

そして、`Cargo.toml`の`[dependencies]`セクションに`sea-orm`を追加します。

```toml
sea-orm = { version = "^0", features = [ <DATABASE_DRIVER>, <ASYNC_RUNTIME>, "macros" ] }
```

`DATABASE_DRIVER`と`ASYNC_RUNTIME`を選択しなければなりません。
もし、SeaORMが生成するエンティティを使用する場合（ほとんどの場合）、`macro`が必要です。

### DATABASE_DRIVER


* `sqlx-mysql` - SQLx MySQL
* `sqlx-postgres` - SQLx PostgreSQL
* `sqlx-sqlite` - SQLx SQLite


### ASYNC_RUNTIME

非同期ランタイムを、以下から1つ選択する。

* `runtime-actix-native-tls`
* `runtime-async-std-native-tls`
* `runtime-tokio-native-tls`
* `runtime-actix-rustls`
* `runtime-async-std-rustls`
* `runtime-tokio-rustls`

基本的に、これらは、`runtime-ASYNC_RUNTIME-TLS_LIB`の形式になっている。

* `ASYNC_RUNTIME`は、`actix`、`async-std`または`tokio`になる。
* `TLS_LIB`は、`native-tls`か`rustls`のどちらかである。

1. `Rust`のWebフレームワークに対応する`ASYNC_RUNTIME`を選択する。

| ASYNC_RUNTIME | Web Framework                                                           |
| ------------- | ----------------------------------------------------------------------- |
| `actix`       | [Actix](https://actix.rs/)                                              |
| `async-std`   | [Tide](https://docs.rs/tide)                                            |
| `tokio`       | [Axum](https://docs.rs/axum/latest/axum/)、[Rocket](https://rocket.rs/) |

2. `native-tls`はプラットフォームのネイティブセキュリティ機能を指定するが、`rustls`は生粋の`Rust`の実装である。

### 追加機能

* `debug-print` - ロガーにすべてのSQL文を出力する。
* `mock` - ユニットテスト用のモックインタフェースである。

## スキーマ管理

テーブルとデータが記録されているデータベースが既にある場合、このセクションをスキップできる。

新規データベースから開始する場合、マイグレーションツールでデータベースのスキーマをバージョン管理することが望ましい。

`SearORM`スキーマ管理ユーティリティで開発するのであれば、`SQLx`の`sqlx-cli`を使用できる。

```bash
cargo install sqlx-cli
```

環境変数に`DATABASE_URL`を設定するか、プロジェクトのルートに`.env`ファイルを作成する。そして、データベース接続設定を記述する。

```.env
DATABASE_URL=sql://username:password@localhost/database
```

新しい`.sql`ファイルを作成する。

```bash
sqlx migrate add <name>
```

マイグレーションを実行する。

```bash
sqlx migrate run
```

### コネクションプール

データベース接続を得るために、[Database](https://docs.rs/sea-orm/0.5/sea_orm/struct.Database.html)インタフェースを使用する。

```rust
let db: DatabaseConnection = Database::connect("protocol://username:password@host/database").await?;
```

`protocol`は、`mysql`、`postgres`または`sqlite`である。

`host`は通常`localhost`で、ドメイン名またはIPアドレスを指定する。

内部で、[sqlx::Pool](https://docs.rs/sqlx/0.5.x/sqlx/struct.Pool.html)が作成され、[DatabaseConnection](https://docs.rs/sea-orm/0.5/sea_orm/enum.DatabaseConnection.html)によって所有される。

`DatabaseConnection`の`execute`または`query_one/all`を呼び出すごとに、プールからコネクションが得られ、解放される。

複数のクエリが、`await`をつけることで並列に実行される。

### 接続オプション

接続を構成するために、[ConnectOptions](https://docs.rs/sea-orm/0.5/sea_orm/struct.ConnectOptions.html)インターフェースを使用する。

```rust
let mut opt = ConnectOptions::new("protocol://username:password@host/database".to_owned());
opt.max_connections(100)
    .min_connections(5)
    .connect_timeout(Duration::from_secs(8))
    .idle_timeout(Duration::from_secs(8))
    .sqlx_logging(true);

let db = Database::connect(opt).await?;
```

### デバッグログ

`debug-print`機能を有効にした場合、`SeaORM`は[tracing](https://crates.io/crates/tracing)クレートによりデバッグメッセージをログに出力する。

デバッグログを捕捉して閲覧するために[tracing-subscriber](https://crates.io/crates/tracing-subscriber)を設定する必要がある。
下記のスニペットと[完全な例](https://github.com/SeaQL/sea-orm/blob/master/examples/actix_example/src/main.rs)を参照しなさい。

```rust
pub async fn main() {
    tracing_subscriber::fmt()
        .with_max_level(tracing::Level::DEBUG)
        .with_test_writer()
        .init();
    // ...
}
```

`SeaORM`から出力されるデバッグログは、以下の通りフィルタできる。

```bash
$ RUST_LOG=debug cargo run 2>&1 | grep sea_orm
[2021-02-02T20:20:20Z DEBUG sea_orm::driver::sqlx_mysql] SELECT `cake`.`id`, `cake`.`name` FROM `cake` LIMIT 1
```
