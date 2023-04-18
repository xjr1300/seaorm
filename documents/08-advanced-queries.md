# 高度なクエリ

- [高度なクエリ](#高度なクエリ)
  - [カスタム選択](#カスタム選択)
    - [部分的な属性の選択](#部分的な属性の選択)
    - [カスタム式の選択](#カスタム式の選択)
    - [選択結果の処理](#選択結果の処理)
      - [カスタム構造体](#カスタム構造体)
      - [構造化されていないタプル](#構造化されていないタプル)
  - [条件式](#条件式)
    - [AND条件](#and条件)
    - [OR条件](#or条件)
    - [ネストした条件](#ネストした条件)
    - [流暢な条件問い合わせ](#流暢な条件問い合わせ)
  - [集計関数](#集計関数)
    - [Group by](#group-by)
    - [Having](#having)
  - [カスタム結合](#カスタム結合)
  - [サブクエリ](#サブクエリ)
    - [サブクエリの条件式](#サブクエリの条件式)
  - [トランザクション](#トランザクション)
    - [クロージャーと共に](#クロージャーと共に)
    - [`begin` \& `commit`/`rollback`](#begin--commitrollback)
    - [ネストしたトランザクション](#ネストしたトランザクション)
    - [分離レベルとアクセスモード](#分離レベルとアクセスモード)
      - [分離レベル](#分離レベル)
      - [アクセスモード](#アクセスモード)
  - [ストリーミング](#ストリーミング)
  - [特別な`ActiveModel`](#特別なactivemodel)

## カスタム選択

デフォルトでは、SeaORMは`Column`列挙型で定義されたすべての列を選択します。
もし、特定の列のみを選択したい場合、そのデフォルトを上書きできます。

```rust
// すべての列を選択します。
assert_eq!(
    cake::Entity::find()
        .build(DbBackend::Postgres)
        .to_string(),
    r#"SELECT "cake"."id", "cake"."name" FROM "cake""#
);
```

### 部分的な属性の選択

`select_ony`メソッドの呼び出しにより、デフォルトの選択をクリアします。
そして、後で、属性のいくつかまたはカスタム式を選択できます。

```rust
// 名前列のみを選択します。
assert_eq!(
    cake::Entity::find()
        .select_only()
        .column(cake::Column::Name)
        .build(DbBackend::Postgres)
        .to_string(),
    r#"SELECT "cake"."name" FROM "cake""#
);
```

もし、1度に複数の属性を選択したい場合、配列で提供できます。

```rust
assert_eq!(
    cake::Entity::find()
        .select_only()
        .columns([cake::Column::Id, cake::Column::Name])
        .build(DbBackend::Postgres)
        .to_string(),
    r#"SELECT "cake"."id", "cake"."name" FROM "cake""#
);
```

高度な例: 特定の列を除いたすべての列の条件選択の例を次に示します。

```rust
assert_eq!(
    cake::Entity::find()
        .select_only()
        .columns(cake::Column::iter().filter(|col| match col {
            cake::Column::Id => false,
            _ => true,
        }))
        .build(DbBackend::Postgres)
        .to_string(),
    r#"SELECT "cake"."name" FROM "cake""#
);
```

### カスタム式の選択

`column_as`メソッドで任意のカスタム式を選択して、それは任意の[sea_query::SimpleExpr](https://docs.rs/sea-query/*/sea_query/expr/enum.SimpleExpr.html)とエイリアスを受け取ります。
[sea_query::Expr](https://docs.rs/sea-query/*/sea_query/expr/struct.Expr.html)ヘルパーは`SimpleExpr`を構築するために使用します。

```rust
use sea_query::{Alias, Expr};

assert_eq!(
    cake::Entity::find()
        .column_as(Expr::col(cake::Column::Id).max().sub(Expr::col(cake::Column::Id)), "id_diff")
        .column_as(Expr::cust("CURRENT_TIMESTAMP"), "current_time")
        .build(DbBackend::Postgres)
        .to_string(),
    r#"SELECT "cake"."id", "cake"."name", MAX("id") - "id" AS "id_diff", CURRENT_TIMESTAMP AS "current_time" FROM "cake""#
);
```

### 選択結果の処理

#### カスタム構造体

複雑な問い合わせの結果を処理するために、`FromQueryResult`トレイトから導出したカスタム構造体を使用できます。
それは、直接モデルに変換できないカスタム列や複数結合を扱うとき、特に役に立ちます。
それは、（ネイティブな）SQLのような任意の問い合わせの結果を受け取るために使用されるかもしれません。

```rust
use sea_orm::{FromQueryResult, JoinType, RelationTrait};
use sea_query::Expr;

#[derive(FromQueryResult)]
struct CakeAndFillingCount {
    id: i32,
    name: String,
    count: i32,
}

let cake_counts: Vec<CakeAndFillingCount> = cake::Entity::find()
    .column_as(filling::Column::Id.count(), "count")
    .join_rev(
        // この場で`RelationDef`を構築します。
        JoinType::InnerJoin,
        cake_filling::Entity::belongs_to(cake::Entity)
            .from(cake_filling::Column::CakeId)
            .to(cake::Column::Id)
            .into()
    )
    // 存在するエンティティから`Relation`を再利用します。
    .join(JoinType::InnerJoin, cake_filling::Relation::Filling.def())
    .group_by(cake::Column::Id)
    .into_model::<CakeAndFillingCount>()
    .all(db)
    .await?;
```

#### 構造化されていないタプル

`into_tuple`メソッドでタプル（または単独の値）を選択できます。

```rust
use sea_orm::{entity::*, query::*, tests_cfg::cake, DeriveColumn, EnumIter};

let res: Vec<(String, i64)> = cake::Entity::find()
    .select_only()
    .column(cake::Column::Name)
    .column(cake::Column::Id.count())
    .group_by(cake::Column::Name)
    .into_tuple()
    .all(&db)
    .await?;
```

## 条件式

`filter`メソッドでSeaORMが検索するために条件を追加できます。
また、`having`メソッドで集計結果を制限することもできます。
それらは両方とも、パラメーターとして[sea_query::Condition](https://docs.rs/sea-query/0.12.7/sea_query/query/struct.Condition.html)を受け取ります。

### AND条件

`Condition::all`メソッドでAND条件式を構築して、`add`メソッドで[sea_query::SimpleExpr](https://docs.rs/sea-query/*/sea_query/expr/enum.SimpleExpr.html)に任意の条件式を追加します。

```rust
assert_eq!(
    cake::Entity::find()
        .filter(
            Condition::all()
                .add(cake::Column::Id.gte(1))
                .add(cake::Column::Name.like("%Cheese%"))
        )
        .build(DbBackend::MySql)
        .to_string(),
    [
        "SELECT `cake`.`id`, `cake`.`name` FROM `cake`",
        "WHERE `cake`.`id` >= 1 AND `cake`.`name` LIKE '%Cheese%'",
    ].join(" ")
);
```

### OR条件

`Condition::any`メソッドでOR条件式を構築して、`add`メソッドで[sea_query::SimpleExpr](https://docs.rs/sea-query/*/sea_query/expr/enum.SimpleExpr.html)に任意の条件式を追加します。

```rust
assert_eq!(
    cake::Entity::find()
        .filter(
            Condition::any()
                .add(cake::Column::Id.eq(4))
                .add(cake::Column::Id.eq(5))
        )
        .build(DbBackend::MySql)
        .to_string(),
    [
        "SELECT `cake`.`id`, `cake`.`name` FROM `cake`",
        "WHERE `cake`.`id` = 4 OR `cake`.`id` = 5",
    ].join(" ")
);
```

### ネストした条件

`add`メソッドは、他の条件式を受け取ることもできます。
これをするために、柔軟な複雑でネストした条件を構築できます。

```rust
assert_eq!(
    cake::Entity::find()
        .filter(
            Condition::any()
                .add(
                    Condition::all()
                        .add(cake::Column::Id.lte(30))
                        .add(cake::Column::Name.like("%Chocolate%"))
                )
                .add(
                    Condition::all()
                        .add(cake::Column::Id.gte(1))
                        .add(cake::Column::Name.like("%Cheese%"))
                )
        )
        .build(DbBackend::MySql)
        .to_string(),
    [
        "SELECT `cake`.`id`, `cake`.`name` FROM `cake`",
        "WHERE (`cake`.`id` <= 30 AND `cake`.`name` LIKE '%Chocolate%') OR",
        "(`cake`.`id` >= 1 AND `cake`.`name` LIKE '%Cheese%')",
    ].join(" ")
);
```

### 流暢な条件問い合わせ

もし、`Option<T>`に`Some<_>`が与えられた場合、`QueryStatement`に操作を適用します。
それは、流暢なクエリ式を維持します。

```rust
use sea_orm::{entity::*, query::*, tests_cfg::cake, DbBackend};

assert_eq!(
    cake::Entity::find()
        .apply_if(Some(3), |mut query, v| {
            query.filter(cake::Column::Id.eq(v))
        })
        .apply_if(Some(100), QuerySelect::limit)
        .apply_if(None, QuerySelect::offset::<Option<u64>>) // オフセットなし
        .build(DbBackend::Postgres)
        .to_string(),
    r#"SELECT "cake"."id", "cake"."name" FROM "cake" WHERE "cake"."id" = 3 LIMIT 100"#
);
```

## 集計関数

SeaORMが検索した結果を`group_by`メソッドでグループ化できます。
もし、グループ化された結果の集合を制限したい場合、`having`メソッドはそれを得ることを支援します。

### Group by

`group_by`メソッドは、エンティティの列または複雑な[sea_query::SimpleExpr](https://docs.rs/sea-query/*/sea_query/expr/enum.SimpleExpr.html)を受け取る。

```rust
assert_eq!(
    cake::Entity::find()
        .select_only()
        .column(cake::Column::Name)
        .group_by(cake::Column::Name)
        .build(DbBackend::Postgres)
        .to_string(),
    r#"SELECT "cake"."name" FROM "cake" GROUP BY "cake"."name""#
);

assert_eq!(
    cake::Entity::find()
        .select_only()
        .column_as(cake::Column::Id.count(), "count")
        .column_as(cake::Column::Id.sum(), "sum_of_id")
        .group_by(cake::Column::Name)
        .build(DbBackend::Postgres)
        .to_string(),
    r#"SELECT COUNT("cake"."id") AS "count", SUM("cake"."id") AS "sum_of_id" FROM "cake" GROUP BY "cake"."name""#
);
```

### Having

`having`メソッドは、前節で紹介した任意の条件式を受け取れます。

```rust
assert_eq!(
    cake::Entity::find()
        .having(cake::Column::Id.eq(4))
        .having(cake::Column::Id.eq(5))
        .build(DbBackend::MySql)
        .to_string(),
    "SELECT `cake`.`id`, `cake`.`name` FROM `cake` HAVING `cake`.`id` = 4 AND `cake`.`id` = 5"
);

assert_eq!(
    cake::Entity::find()
        .select_only()
        .column_as(cake::Column::Id.count(), "count")
        .column_as(cake::Column::Id.sum(), "sum_of_id")
        .group_by(cake::Column::Name)
        .having(cake::Column::Id.gt(6))
        .build(DbBackend::MySql)
        .to_string(),
    "SELECT COUNT(`cake`.`id`) AS `count`, SUM(`cake`.`id`) AS `sum_of_id` FROM `cake` GROUP BY `cake`.`name` HAVING `cake`.`id` > 6"
);
```

> ℹ️ INFO
>
> `max`、`min`、`sum`、`count`のような集計関数は、[ColumnTrait](https://docs.rs/sea-orm/*/sea_orm/entity/prelude/trait.ColumnTrait.html)で利用できます。

## カスタム結合

複雑な結合選択問い合わせを構築するために`join`メソッドを使用できます。
`join`メソッドはエンティティファイルで定義された任意の`RelationDef`を受け取り、その上、`belong_to`メソッドで関連を定義できます。
内部結合、左外部結合および右外部結合のような結合の種類は、`JoinType`を使用することで指定されます。

```rust
use sea_orm::{JoinType, RelationTrait};
use sea_query::Expr;

assert_eq!(
    cake::Entity::find()
        .column_as(filling::Column::Id.count(), "count")
        .column_as(
            Expr::tbl(Alias::new("fruit_alias"), fruit::Column::Name).into_simple_expr(),
            "fruit_name"
        )
        // この場で`RelationDef`を構築します。
        .join_rev(
            JoinType::InnerJoin,
            cake_filling::Entity::belongs_to(cake::Entity)
                .from(cake_filling::Column::CakeId)
                .to(cake::Column::Id)
                .into()
        )
        // 存在するエンティティから`Relation`を再利用します。
        .join(JoinType::InnerJoin, cake_filling::Relation::Filling.def())
        // カスタム条件でテーブルエイリアスと結合します。
        .join_as(
            JoinType::LeftJoin,
            cake::Relation::Fruit
                .def()
                .on_condition(|_left, right| {
                    Expr::tbl(right, fruit::Column::Name)
                        .like("%tropical%")
                        .into_condition()
                }),
            Alias::new("fruit_alias")
        )
        .group_by(cake::Column::Id)
        .having(filling::Column::Id.count().equals(Expr::value(2)))
        .build(DbBackend::MySql)
        .to_string(),
    [
        "SELECT `cake`.`id`, `cake`.`name`, COUNT(`filling`.`id`) AS `count`, `fruit_alias`.`name` AS `fruit_name` FROM `cake`",
        "INNER JOIN `cake_filling` ON `cake_filling`.`cake_id` = `cake`.`id`",
        "INNER JOIN `filling` ON `cake_filling`.`filling_id` = `filling`.`id`",
        "LEFT JOIN `fruit` AS `fruit_alias` ON `cake`.`id` = `fruit_alias`.`cake_id` AND `fruit_alias`.`name` LIKE '%tropical%'",
        "GROUP BY `cake`.`id`",
        "HAVING COUNT(`filling`.`id`) = 2",
    ]
    .join(" ")
);
```

複雑なクエリの結果を操作するために、`FromQueryResult`トレイトから導出したカスタム構造体を使用できます。
詳細は[ここ](https://www.sea-ql.org/SeaORM/docs/advanced-query/custom-select#handling-custom-selects)を参照してください。

## サブクエリ

### サブクエリの条件式

サブクエリで条件式を構築するために、`in_subquery`または`not_in_subquery`メソッドを使用します。

```rust
use sea_orm::Condition;
use sea_query::Query;

assert_eq!(
    cake::Entity::find()
        .filter(
            Condition::any().add(
                cake::Column::Id.in_subquery(
                    Query::select()
                        .expr(cake::Column::Id.max())
                        .from(cake::Entity)
                        .to_owned()
                )
            )
        )
        .build(DbBackend::MySql)
        .to_string(),
    [
        "SELECT `cake`.`id`, `cake`.`name` FROM `cake`",
        "WHERE `cake`.`id` IN (SELECT MAX(`cake`.`id`) FROM `cake`)",
    ]
    .join(" ")
);
```

## トランザクション

トランザクションは、ACIDを保証しながら実行されるSQL文のグループです。
2つのトランザクションAPIがあります。

### クロージャーと共に

[クロージャーでトランザクション](https://docs.rs/sea-orm/*/sea_orm/trait.TransactionTrait.html#tymethod.transaction)を実行できます。
トランザクションは、クロージャーが`Ok`を返却した場合にコミットされ、`Err`を返却した場合にロールバックされます。
2番目と3番目の型パラメーターは、それぞれ`Ok`と`Err`型です。
`async_closure`は未だ安定化されていないため、それを`Pin<Box<>_>>`する必要があります。

```rust
// <Fn, A, B> -> Result<A, B>
db.transaction::<_, (), DbErr>(|txn| {
    Box::pin(async move {
        bakery::ActiveModel {
            name: Set("SeaSide Bakery".to_owned()),
            profit_margin: Set(10.4),
            ..Default::default()
        }
        .save(txn)
        .await?;

        bakery::ActiveModel {
            name: Set("Top Bakery".to_owned()),
            profit_margin: Set(15.0),
            ..Default::default()
        }
        .save(txn)
        .await?;

        Ok(())
    })
})
.await;
```

これは多くのケースで好ましい方法です。
しかし、もし非同期ブロック内で参照をキャプチャーを試みているが、*不可能なライフタイム*に遭遇した場合は、次のAPIが解決さくです。

### `begin` & `commit`/`rollback`

`begin`がトランザクションを開始して、`commit`または`rollback`がそれに続きます。
もし、`txn`がスコープを外れた場合、そのトランザクションは自動的にロールバックされます。

```rust
let txn = db.begin().await?;

bakery::ActiveModel {
    name: Set("SeaSide Bakery".to_owned()),
    profit_margin: Set(10.4),
    ..Default::default()
}
.save(&txn)
.await?;

bakery::ActiveModel {
    name: Set("Top Bakery".to_owned()),
    profit_margin: Set(15.0),
    ..Default::default()
}
.save(&txn)
.await?;

txn.commit().await?;
```

### ネストしたトランザクション

ネストしたトランザクションは、データベースの`SAVEPOINT`で実行されています。
下の例は、クロージャーAPIで振る舞いを説明しています。

```rust
assert_eq!(Bakery::find().all(txn).await?.len(), 0);

ctx.db.transaction::<_, _, DbErr>(|txn| {
    Box::pin(async move {
        let _ = bakery::ActiveModel {..}.save(txn).await?;
        let _ = bakery::ActiveModel {..}.save(txn).await?;
        assert_eq!(Bakery::find().all(txn).await?.len(), 2);

        // ネストしたトランザクションをコミットすることを試みます。
        txn.transaction::<_, _, DbErr>(|txn| {
            Box::pin(async move {
                let _ = bakery::ActiveModel {..}.save(txn).await?;
                assert_eq!(Bakery::find().all(txn).await?.len(), 3);

                // ネストしたトランザクションをロールバックすることを試みます。
                assert!(txn.transaction::<_, _, DbErr>(|txn| {
                        Box::pin(async move {
                            let _ = bakery::ActiveModel {..}.save(txn).await?;
                            assert_eq!(Bakery::find().all(txn).await?.len(), 4);

                            Err(DbErr::Query(RuntimeErr::Internal(
                                "Force Rollback!".to_owned(),
                            )))
                        })
                    })
                    .await
                    .is_err()
                );

                assert_eq!(Bakery::find().all(txn).await?.len(), 3);

                // 二重にネストしたトランザクションをコミットすることを試みます。
                txn.transaction::<_, _, DbErr>(|txn| {
                    Box::pin(async move {
                        let _ = bakery::ActiveModel {..}.save(txn).await?;
                        assert_eq!(Bakery::find().all(txn).await?.len(), 4);

                        Ok(())
                    })
                })
                .await;

                assert_eq!(Bakery::find().all(txn).await?.len(), 4);

                Ok(())
            })
        })
        .await;

        Ok(())
    })
})
.await;

assert_eq!(Bakery::find().all(txn).await?.len(), 4);
```

### 分離レベルとアクセスモード

`0.10.5`に導入された、[transaction_with_config](https://docs.rs/sea-orm/*/sea_orm/trait.TransactionTrait.html#tymethod.transaction_with_config)と[begin_with_config](https://docs.rs/sea-orm/*/sea_orm/trait.TransactionTrait.html#tymethod.begin_with_config)は、[分離レベル](https://docs.rs/sea-orm/*/sea_orm/enum.IsolationLevel.html)と[アクセスモード](https://docs.rs/sea-orm/*/sea_orm/enum.AccessMode.html)を指定できるようにします。

#### 分離レベル

`RepeatableRead`: 同じトランザクション内で一貫性のある読み取りは、最初の読み込みによって確立されたスナップショットを読み込みます。

`ReadCommitted`: 同じトランザクション内であっても、一般性のある読み取りごとに、それ独自の新しいスナップショットが作成されて読み込みます。

`ReadUncommitted`: SELECT文がブロックされずに実行されますが、前のバージョンの行が使用される可能性があります。

`Serializable`: 現在のトランザクションのすべての文が、最初のクエリの前にコミットされた行のみ、またはこのトランザクション内でデータを変更する文が実行した結果を見ることができます。

#### アクセスモード

`ReadOnly`: データは、このトランザクション内で変更されません。

`ReadWrite`: データは、このトランザクジョン内で変更されます（デフォルト）。

## ストリーミング

メモリ割り当てを減らして、効率を改善するために、任意の選択で[futures](https://crates.io/crates/futures)クレートの非同期ストリームを使用します。

```rust
use futures::TryStreamExt;

// すべてのフルーツをストリーミングします。
let mut stream = Fruit::find().stream(db).await?;

while let Some(item) = stream.try_next().await? {
    let item: fruit::ActiveModel = item.into();
    // itemで何かします。
}
```

```rust
// 名前に文字"a"を含むすべてのフルーツをストリーミングします。
let mut stream = Fruit::find()
    .filter(fruit::Column::Name.contains("a"))
    .order_by_asc(fruit::Column::Name)
    .stream(db)
    .await?;
```

ストリームオブジェクトはドロップされるまで排他的にコネクションを保持するため、他によってコネクションが借用されることを防ぐ必要があることに注意してください。

```rust
{
    let s1 = Fruit::find().stream(db).await?;
    let s2 = Fruit::find().stream(db).await?;
    let s3 = Fruit::find().stream(db).await?;
    // 3つのコネクションが保持されています。
}
// すべてのストリームがドロップされ、コネクションはコネクションプールに戻されます。
```

## 特別な`ActiveModel`

`into_active_model`メソッドで`ActiveModel`に変換できる、モデルの部分的なフィールドを持つ`IntoActiveModel`を実装をもつ構造体を作成する。
例えば、その構造体はREST APIにおいてフォーム送信として使用される。

`IntoActiveValue`トレイトは、`into_active_value`メソッドで`Option<T>`を`ActiveValue<T>`に変換する。

```rust
// Define regular model as usual
#[derive(Clone, Debug, PartialEq, DeriveEntityModel)]
#[sea_orm(table_name = "fruit")]
pub struct Model {
    #[sea_orm(primary_key)]
    pub id: i32,
    pub name: String,
    pub cake_id: Option<i32>,
}
```

いくつかのフィールドを省略した新しい構造体を作成する。

```rust
use sea_orm::ActiveValue::NotSet;

#[derive(DeriveIntoActiveModel)]
pub struct NewFruit {
    // id is omitted
    pub name: String,
    // it is required as opposed to optional in Model
    pub cake_id: i32,
}

assert_eq!(
    NewFruit {
        name: "Apple".to_owned(),
        cake_id: 1,
    }
    .into_active_model(),
    fruit::ActiveModel {
        id: NotSet,
        name: Set("Apple".to_owned()),
        cake_id: Set(Some(1)),
    }
);
```

`Option<Option<T>>`は、`Some(None)`によって列をNULLに更新する。

```rust
use sea_orm::ActiveValue::NotSet;

#[derive(DeriveIntoActiveModel)]
pub struct UpdateFruit {
    pub cake_id: Option<Option<i32>>,
}

assert_eq!(
    UpdateFruit {
        cake_id: Some(Some(1)),
    }
    .into_active_model(),
    fruit::ActiveModel {
        id: NotSet,
        name: NotSet,
        cake_id: Set(Some(1)),
    }
);

assert_eq!(
    UpdateFruit {
        cake_id: Some(None),
    }
    .into_active_model(),
    fruit::ActiveModel {
        id: NotSet,
        name: NotSet,
        cake_id: Set(None),
    }
);

assert_eq!(
    UpdateFruit {
        cake_id: None,
    }
    .into_active_model(),
    fruit::ActiveModel {
        id: NotSet,
        name: NotSet,
        cake_id: NotSet,
    }
);
```
