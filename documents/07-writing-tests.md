# テストの記述

- [テストの記述](#テストの記述)
  - [強固と性格](#強固と性格)
    - [エラーの型](#エラーの型)
      - [1. 型のエラー](#1-型のエラー)
      - [2. トランザクションエラー](#2-トランザクションエラー)
      - [3. 振る舞いのエラー](#3-振る舞いのエラー)
  - [軽減策](#軽減策)
    - [1. 型のエラー](#1-型のエラー-1)
    - [2. トランザクションエラー](#2-トランザクションエラー-1)
    - [3. 振る舞いのエラー](#3-振る舞いのエラー-1)
  - [モックインターフェース](#モックインターフェース)
    - [クエリ結果のモック](#クエリ結果のモック)
    - [実行結果のモック](#実行結果のモック)
  - [SQLiteの使用](#sqliteの使用)
    - [統合テスト](#統合テスト)
    - [データベーススキーマの準備](#データベーススキーマの準備)
    - [テストの実行](#テストの実行)

## 強固と性格

テストは、Rustのプログラムでなくてはならない部分です。
[cargo test](https://doc.rust-lang.org/cargo/commands/cargo-test.html)がビルトインされていることを確認してください。

もし、`unsafe`を使用しておらず、コードがコンパイルされれば、そのRustプログラムは安全です。
しかし、自動的に*強固*になる訳ではありません。
もし、エラー処理を注意深くしていない場合、依然としてプログラムは予期せずパニックする可能性があります。

プログラムがパニックしなかったとしても、それはそれが*正確*であることを意味していません。
依然として、それは誤った振る舞いをして、混沌としたデータを作成する可能性があります。

適切なテストを記述することにより、プログラムの正確性を改善できます。

### エラーの型

まず、データ駆動アプリケーションにおける、エラーの原因の違いを分類します。

#### 1. 型のエラー

1. スペルミスまたは存在しないシンボル（テーブルまたは列）の名前
2. 互換性のない関数またはデータ操作（例えば、2つの文字列を加算する）の使用
3. 不正なSQLクエリ

不正確なSQLクエリには、次の例があります。

- 例えば、`JOIN`クエリ内の不明確なシンボル
- 例えば、非NULL許容列にデータを挿入することを忘れる
- 例えば、`GROUP BY`クエリ内でそれぞれの列を集めることを忘れる

#### 2. トランザクションエラー

1. エンティティの関連の維持に失敗
2. データの一貫性と制約の維持に失敗

#### 3. 振る舞いのエラー

1. 誤った条件で結合またはフィルタする
2. 不完全または誤ったクエリの結果
3. 予期しない副作用を伴った挿入、更新または削除操作
4. 意図しない何らかの振る舞い

> '予期しない副作用'に関する注意事項: 関連に厳密な親子関係がない場合に`CASCADE`を使用してはいけません。

## 軽減策

ここで、どのようにこれらのエラーを軽減できるか確認します。

### 1. 型のエラー

Rustを使用することで、自動的にシンボルのミススペルから守れます。

*完全に静的な*クエリビルダーを使用することは、この種類のエラーを完全に取り除くことができます。
しかし、それは、それぞれのパラメーターが静的に定義されていて、コンパイル時に利用できることを必要とします。
プログラムが開始（環境変数）してから、実行（ランタイムな設定の変更）するまで、知ることができないことが常にあるため、これは*厳しい*要件です。
型システムが常に動的なスクリプト言語をバックグラウンドに持つ開発者の場合、これは特に扱いにくいです。

そのような理由で、SeaORMはコンパイル時にチェックすることを試みません。

単体テストで有効で、プロダクションで無効な言及した問題に対して、動的に生成されたクエリをランタイムでリンティングする機能を提供することを意図しています（まだ開発中ですが）。

### 2. トランザクションエラー

これらの問題は取り除くことができません。
通常、それはコードに何らかの論理バグがあることを示しています。
それらが発生したとき、それは既に遅く、選択肢が中断することしかありません。
代わりに、データの操作を試みる前に一貫性を確認して、それらを積極的に回避する必要があります。

誤ったデータを拒否して、データベースに記録されることを回避できる単体テストの集まりを記述するべきです。
単体テストは、それぞれの*トランザクション*（アプリケーションドメインで、必ずしもデータベーストランザクションではありません）が正しいか検証もするべきです。

SeaORMは、`Mock`データベースインターフェースを使用したこれらの単体テストを記述することを支援します。

### 3. 振る舞いのエラー

これは、基本的にドメインレベルにおけるプログラム全体のテストで、データを与えて一般的なユーザーの操作を提供する必要があります。
通常、実際のデータベースに対するCIでそれを実行します。
しかし、SeaORMはこれらのテストの大きさを小さくするため、最も重要なデータの流れは、Cargoの[統合テスト](https://doc.rust-lang.org/rust-by-example/testing/integration_testing.html)によってテストできます。

SeaORMは、MySQL、PostgreSQL及びSQLiteを抽象化するため、プログラムの振る舞いをテストするバックエンドとしてSQLiteを使用できます。
SQLiteは、それを頻繁に、ローカルで、CIで実行するために十分なほど軽量です。

問題は、SQLiteがMySQLまたはPostgreSQLの高度な機能のいくつかを欠いていることで、データベース特有の機能の使用に依存して、すべてのロジックがSQLiteでテストできる訳ではありません。

MySQLとPostgreSQLのより高度な機能をシミュレートできるSQLiteの代わりをさがしているところです。

## モックインターフェース

モックデータベースインターフェースを使用して、アプリケーションロジックをユニットテストできる。

モックデータベースはデータを持たないため、CRUD操作を実行したときに返却される予期されたデータを定義する必要がある。

- `query result`は選択操作を提供しなければならない。
- `exec result`は挿入、更新及び削除操作をサポートする機能を提供しなければならない。

アプリケーションロジックが正しいことを確信するために、モックデータベースでトランザクションログを検証することができる。

モックデータベースへの接続を使用してユニットテストを記述する方法は、[ここ](https://github.com/SeaQL/sea-orm/blob/master/src/executor/paginator.rs#L250)にある。

### クエリ結果のモック

`MockDatabase::new(DatabaseBackend::Postgres)`でPostgreSQL用のモックデータベースを作成する。
クエリ結果は、`append_query_results`メソッドを使用して準備する。
複数のクエリ結果を表現するために、ベクタのベクタを渡していることに注意すること。ベクタ内の各ベクタは、1つ以上のモデルを含んでいる。
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
