# テストの記述

## 強固で正確なシステム

Rustにおいて、テストはプログミングの必須なバーツである。
[cargo test](https://doc.rust-lang.org/cargo/commands/cargo-test.html)ビルトインを参照できる。

`unsafe`を使用しておらず、コードがコンパイルされれば、そのRustプログラムは安全である。
しかしながら、自動的に強固になならない。
エラハンドリングに注意していなければ、プログラムは予期せずパニックを起こす。

プログラムがパニックを起こさなくても、正しいことを意味していない。
プログラムは間違った振る舞いを行い、混沌としたデータを作成する。

適切なテストを記述することによって、プログラムを正しく修正できる。

## エラーの種類

まず、データ駆動アプリケーションにおけるエラーの原因の違いを分類する。

### 1. 型のエラー

1. スペルミスまたはテーブルやカラムなど存在しないシンボルの名前。
2. 2つの文字列を加えるなど、互換性のない関数や操作の使用。
3. 不正なSQLクエリ
   * `JOIN`クエリ内の不明確なシンボル。
   * 必須カラムに対するデータの挿入忘れ。
   * `GROUP BY`クエリ内のすべての列の集合忘れ。

### 2. トランザクションエラー

1. エンティティのリレーションシップの維持の失敗。
2. データの一貫性と制約の維持の失敗。

### 3. 振る舞いのエラー

1. 間違った条件を使用した結合やフィルタリング。
2. 不完全または間違ったクエリの結果。
3. 予期しない副作用を伴った挿入、更新または削除操作。
4. 意図しない何らかの振る舞い。

:::note info
'予期しない副作用': 強い親と子の関係のない`CASCADE`を使用してはならない。
:::

## 軽減策

どのようにこれらのエラーを軽減するかを説明する。

### 1. 型のエラー

Rustを使用することで、自動的にスペルミスしたシンボルから守ることができる。

`Diesel`のような完全な静的クエリビルダーを使用することで、この種類のエラーを排除できる。
しかしながら、すべてのパラメータが静的に定義されて、コンパイル時に存在する必要がある。
これが厳しい要求である理由は、プログラムが開始するまで環境変数がわからず、プログラムの実行中は実行時に設定が変更されるためである。
型システムが常に動的なスクリプト言語からきたプログラマーにとって、これは特に厄介である。

そのような理由で、SeaORMはコンパイル時にチェックすることを試みない。
まだ開発中であるが、単体テストで有効にして本番環境で無効にできる、前述の問題に対して動的に生成されたクエリを実行時にリンティングする機能を提供することを意図している。

### 2. トランザクションエラー

トランザクションエラーの問題は排除できない。
通常、トランザクションエラーは、コードにいくつかの論理的なバグがあることを意味している。
トランザクションエラーが発生したとき、もはや遅く、唯一の選択は中断である。
代わりに、トランザクションエラーは積極的に避ける必要があり、データの操作を試みる前に、前もって制約を確認する。

不正なデータを排除してデータベースに記録されることを防ぐような多くのユニットテストを記述する必要がある。
ユニットテストは、各トランザクション（アプリケーションドメインで必ずデータベーストランザクションである必要はない）が正常であることを確認する必要がある。

SeaORMは、`Mock`データベースインターフェースを使用することで、トランザクションエラーのユニットテストの記述を支援する。

### 3. 振る舞いのエラー

ドメインレベルな基本的な全体プログラムのテストで、生成したデータと一般的なユーザーの操作を提供するこを要求する。
通常、実際のデータベースに対して`CI`を行う。
しかしながら、SeaORMは、これらのテストをスケールダウンして、Cargoの[統合テスト](https://doc.rust-lang.org/rust-by-example/testing/integration_testing.html)で最も重要なデータフローをテストすることを推奨する。

SeaORMはMySQL、PostgreSQL及びSQLiteを抽象化するため、プログラムの振る舞いをテストするバックエンドとしてSQLiteを使用できる。
SQLiteは、頻繁に実行して、ローカルで、`CI`するために十分軽量である。

問題は、SQLiteにはMySQLまたはPostgreSQLの高度な機能がいくつか欠けているため、データベース固有の機能に依存状況によっては、SQLite内ですべてのロジックをテストできるわけではないことである。

MySQLとPostgreSQLの先進的な機能をシミュレートできるSQLite互換を探している。

## モックインターフェース

モックデータベースインターフェースを使用して、アプリケーションロジックをユニットテストできる。

モックデータベースはデータを持たないため、CRUD操作を実行したときに返却される予期されたデータを定義する必要がある。

* `query result`は選択操作を提供しなければならない。
* `exec result`は挿入、更新及び削除操作をサポートする機能を提供しなければならない。

アプリケーションロジックが正しいことを確信するために、モックデータベースでトランザクションログを検証することができる。

モックデータベースへの接続を使用してユニットテストを記述する方法は、[ここ](https://github.com/SeaQL/sea-orm/blob/master/src/executor/paginator.rs#L250)にある。

### クエリ結果のモック

`MockDatabase::new(DatabaseBackend::Postgres)`でPostgreSQL用のモックデータベースを作成する。
クエリ結果は、`append_query_results`メソッドを使用して準備する。
複数のクエリ結果を表現するために、ベクタのベクタを渡していることに注意すること。ベクタ内の各ベクタは、1つより多いモデルを含んでいる。
最終的に、モックデータベースを接続に変換して、通常の実際の接続のようにCRUD操作を実行するために使用する。

`MockDatabase`について1つ特別なことは、モックデータベースのトランザクションログをチェックできることである。
モックデータベースで実行されたSQLクエリは記録され、アプリケーションロジックの正確性を確信するために、それぞれのログを検証できる。

```rust
#[cfg(test)]
mod tests {
    use sea_orm::{
        entity::prelude::*, entity::*, tests_cfg::*,
        DatabaseBackend, MockDatabase, Transaction,
    };

    #[async_std::test]
    async fn test_find_cake() -> Result<(), DbErr> {
        // Create MockDatabase with mock query results
        let db = MockDatabase::new(DatabaseBackend::Postgres)
            .append_query_results(vec![
                // First query result
                vec![cake::Model {
                    id: 1,
                    name: "New York Cheese".to_owned(),
                }],
                // Second query result
                vec![
                    cake::Model {
                        id: 1,
                        name: "New York Cheese".to_owned(),
                    },
                    cake::Model {
                        id: 2,
                        name: "Chocolate Forest".to_owned(),
                    },
                ],
            ])
            .into_connection();

        // Find a cake from MockDatabase
        // Return the first query result
        assert_eq!(
            cake::Entity::find().one(&db).await?,
            Some(cake::Model {
                id: 1,
                name: "New York Cheese".to_owned(),
            })
        );

        // Find all cakes from MockDatabase
        // Return the second query result
        assert_eq!(
            cake::Entity::find().all(&db).await?,
            vec![
                cake::Model {
                    id: 1,
                    name: "New York Cheese".to_owned(),
                },
                cake::Model {
                    id: 2,
                    name: "Chocolate Forest".to_owned(),
                },
            ]
        );

        // Checking transaction log
        assert_eq!(
            db.into_transaction_log(),
            vec![
                Transaction::from_sql_and_values(
                    DatabaseBackend::Postgres,
                    r#"SELECT "cake"."id", "cake"."name" FROM "cake" LIMIT $1"#,
                    vec![1u64.into()]
                ),
                Transaction::from_sql_and_values(
                    DatabaseBackend::Postgres,
                    r#"SELECT "cake"."id", "cake"."name" FROM "cake""#,
                    vec![]
                ),
            ]
        );

        Ok(())
    }
}
```

### 実行結果のモック

実行結果のモックは、クエリ結果のモックととても似ており、違いは`append_exec_results`メソッドを使用して、ユニットテストで挿入、更新及び削除操作を実行する。
`append_exec_results`メソッドは`MockExecResult`のベクタを受け取り、各々は対応する操作の実行結果を表現する。

```rust
#[cfg(test)]
mod tests {
    use sea_orm::{
        entity::prelude::*, entity::*, tests_cfg::*,
        DatabaseBackend, MockDatabase, MockExecResult, Transaction,
    };

    #[async_std::test]
    async fn test_insert_cake() -> Result<(), DbErr> {
        // Create MockDatabase with mock execution result
        let db = MockDatabase::new(DatabaseBackend::Postgres)
            .append_query_results(vec![
                vec![cake::Model {
                    id: 15,
                    name: "Apple Pie".to_owned(),
                }],
                vec![cake::Model {
                    id: 16,
                    name: "Apple Pie".to_owned(),
                }],
            ])
            .append_exec_results(vec![
                MockExecResult {
                    last_insert_id: 15,
                    rows_affected: 1,
                },
                MockExecResult {
                    last_insert_id: 16,
                    rows_affected: 1,
                },
            ])
            .into_connection();

        // Prepare the ActiveModel
        let apple = cake::ActiveModel {
            name: Set("Apple Pie".to_owned()),
            ..Default::default()
        };

        // Insert the ActiveModel into MockDatabase
        assert_eq!(
            apple.clone().insert(&db).await?,
            cake::Model {
                id: 15,
                name: "Apple Pie".to_owned()
            }
        );

        // If you want to check the last insert id
        let insert_result = cake::Entity::insert(apple).exec(&db).await?;
        assert_eq!(insert_result.last_insert_id, 16);

        // Checking transaction log
        assert_eq!(
            db.into_transaction_log(),
            vec![
                Transaction::from_sql_and_values(
                    DatabaseBackend::Postgres,
                    r#"INSERT INTO "cake" ("name") VALUES ($1) RETURNING "id", "name""#,
                    vec!["Apple Pie".into()]
                ),
                Transaction::from_sql_and_values(
                    DatabaseBackend::Postgres,
                    r#"INSERT INTO "cake" ("name") VALUES ($1) RETURNING "id""#,
                    vec!["Apple Pie".into()]
                ),
            ]
        );

        Ok(())
    }
}
```

## SQLiteの使用

データベース特有の機能を必要としないアプリケーションロジックをテストしたい場合、モックデータベースにSQLiteを使用することは良い選択である。

[ここ](https://github.com/SeaQL/sea-orm/blob/master/tests/basic.rs)の簡単な例がある。

### 統合テスト

[統合テスト](https://doc.rust-lang.org/rust-by-example/testing/integration_testing.html)で、より複雑なテストケースを実行することを推奨する。
以下のコードの断片は、データベースへの接続、スキーマの準備及びテストの実行ステップを説明している。

```rust
async fn main() -> Result<(), DbErr> {
    // Connecting SQLite
    let db = Database::connect("sqlite::memory:").await?;

    // Setup database schema
    setup_schema(&db).await?;

    // Performing tests
    testcase(&db).await?;

    Ok(())
}
```

### データベーススキーマの準備

テストするためにSQLiteデータベースのテーブルを作成するために、[TableCreateStatement](https://docs.rs/sea-query/*/sea_query/table/struct.TableCreateStatement.html)を手動で記述する代わりに、[Schema::create_table_from_entity](https://docs.rs/sea-orm/0.5/sea_orm/schema/struct.Schema.html#method.create_table_from_entity)メソッドを使用して`Entity`から派生させることができる。

```rust
async fn setup_schema(db: &DbConn) {

    // Setup Schema helper
    let schema = Schema::new(DbBackend::Sqlite);

    // Derive from Entity
    let stmt: TableCreateStatement = schema.create_table_from_entity(MyEntity);

    // Or setup manually
    assert_eq!(
        stmt.build(SqliteQueryBuilder),
        Table::create()
            .table(MyEntity)
            .col(
                ColumnDef::new(MyEntity::Column::Id)
                    .integer()
                    .not_null()
            )
            //...
            .build(SqliteQueryBuilder)
    );

    // Execute create table statement
    let result = db
        .execute(db.get_database_backend().build(&stmt))
        .await;
}
```

### テストの実行

テストケースを実行して結果に対して確認する。

```rust
async fn testcase(db: &DbConn) -> Result<(), DbErr> {

    let baker_bob = baker::ActiveModel {
        name: Set("Baker Bob".to_owned()),
        contact_details: Set(serde_json::json!({
            "mobile": "+61424000000",
            "home": "0395555555",
            "address": "12 Test St, Testville, Vic, Australia"
        })),
        bakery_id: Set(2),
        ..Default::default()
    };

    let baker_insert_res = Baker::insert(baker_bob)
        .exec(db)
        .await
        .expect("could not insert baker");

    assert_eq!(baker_insert_res.last_insert_id, 1);

    Ok(())
}
```
