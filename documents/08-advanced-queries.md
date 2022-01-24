# 高度なクエリ

## 特殊な選択

デフォルトでは、SeaORMは`Column`列挙型で定義されたすべての列を選択する。
本当に欲しい列のみを選択して、デフォルトの振る舞いを上書きできる。

### デフォルトの選択の説明

必要であれば、`select_only`メソッドを呼び出して、デフォルトの選択を説明する。
そして、いくつかの属性や特別な式を選択できることを、後で説明する。

```rust
// Selecting all columns
assert_eq!(
    cake::Entity::find()
        .build(DbBackend::Postgres)
        .to_string(),
    r#"SELECT "cake"."id", "cake"."name" FROM "cake""#
);
```

### いくつかの属性のみ選択する

`select_only`メソッドと`column`メソッドを使用することで、望んだ属性のみを選択できる。

```rust
// Selecting the name column only
assert_eq!(
    cake::Entity::find()
        .select_only()
        .column(cake::Column::Name)
        .build(DbBackend::Postgres)
        .to_string(),
    r#"SELECT "cake"."name" FROM "cake""#
);
```

### 特別な式を選択する

`column_as`メソッドでいくつかの特別な式を選択して、`sea_query::SimpleExpr`とエイリアスを得られる。
`SimpleExpr`を構築するために`sea_query::Expr`ヘルパーを使用する。

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

### 特別な選択を操作する

複雑なクエリの結果を操作するために、`FromQueryResult`トレイトから派生した特別な構造体を使用できる。
直接モデルに変換できない特別な列や複数の結合を扱うときに特に役立つ。
その特別な構造体は、クエリやSQLの結果を受け取るときに使用される。

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
        // construct `RelationDef` on the fly
        JoinType::InnerJoin,
        cake_filling::Entity::belongs_to(cake::Entity)
            .from(cake_filling::Column::CakeId)
            .to(cake::Column::Id)
            .into()
    )
    // reuse a `Relation` from existing Entity
    .join(JoinType::InnerJoin, cake_filling::Relation::Filling.def())
    .group_by(cake::Column::Id)
    .into_model::<CakeAndFillingCount>()
    .all(db)
    .await?;
```

## 条件式

`filter`メソッドでSeaORMが検索する条件を追加できる。
また、`having`メソッドを使用して集計を制限できる。
それら両方は、パラメータとして[sea_query::Condition](https://docs.rs/sea-query/0.12.7/sea_query/query/struct.Condition.html)を受け取る。

### AND条件

`Condition::all`メソッドでAND条件式を構築して、`add`メソッドで[sea_query::SimpleExpr](https://docs.rs/sea-query/*/sea_query/expr/enum.SimpleExpr.html)に何らかの条件式を追加する。

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

`Condition::any`メソッドでAND条件式を構築して、`add`メソッドで[sea_query::SimpleExpr](https://docs.rs/sea-query/*/sea_query/expr/enum.SimpleExpr.html)に何らかの条件式を追加する。


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

`add`メソッドは、他の条件式を受け取ることができる。
これを実施するために、柔軟な複雑なネストした条件を構築できる。

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

## 集計関数

SeaORMが検索した結果を`group_by`メソッドでグループ化できる。
グループ化された結果セットを制限したい場合は、`having`メソッドでできる。

### Group by（グループ化）

`group_by`めwソッドは、列または[sea_query::SimpleExpr](https://docs.rs/sea-query/*/sea_query/expr/enum.SimpleExpr.html)を受け取る。

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
```

### Having(グループ化した結果のフィルタ)

`having`メソッドは、前のセクションで紹介した何らかの条件式を受け取る。

```rust
assert_eq!(
    cake::Entity::find()
        .having(cake::Column::Id.eq(4))
        .having(cake::Column::Id.eq(5))
        .build(DbBackend::MySql)
        .to_string(),
    "SELECT `cake`.`id`, `cake`.`name` FROM `cake` HAVING `cake`.`id` = 4 AND `cake`.`id` = 5"
);
```

## 特殊な結合

複雑な結合選択クエリを構築するために`join`メソッドを使用できる。
`join`メソッドはエンティティファイルで定義された何らかの`RelationDef`を受け取り、`belong_to`メソッドでリレーションを定義できる。
内部結合、左外部結合および右外部結合のような結合の種類は、`JoinType`を使用することで記述する。

```rust
use sea_orm::{JoinType, RelationTrait};
use sea_query::Expr;

assert_eq!(
    cake::Entity::find()
        .column_as(filling::Column::Id.count(), "count")
        .join_rev(
            // construct `RelationDef` on the fly
            JoinType::InnerJoin,
            cake_filling::Entity::belongs_to(cake::Entity)
                .from(cake_filling::Column::CakeId)
                .to(cake::Column::Id)
                .into()
        )
        // reuse a `Relation` from existing Entity
        .join(JoinType::InnerJoin, cake_filling::Relation::Filling.def())
        .group_by(cake::Column::Id)
        .having(filling::Column::Id.count().equals(Expr::value(2)))
        .build(DbBackend::MySql)
        .to_string(),
    [
        "SELECT `cake`.`id`, `cake`.`name`, COUNT(`filling`.`id`) AS `count` FROM `cake`",
        "INNER JOIN `cake_filling` ON `cake_filling`.`cake_id` = `cake`.`id`",
        "INNER JOIN `filling` ON `cake_filling`.`filling_id` = `filling`.`id`",
        "GROUP BY `cake`.`id`",
        "HAVING COUNT(`filling`.`id`) = 2",
    ]
    .join(" ")
);
```

複雑なクエリの結果を操作するために`FromQueryResult`トレイトから派生した特殊の構造体を使用できる。
詳細は[ここ](https://www.sea-ql.org/SeaORM/docs/advanced-query/custom-select#handling-custom-selects)を参照すること。

## サブクエリ

### サブクエリにおける条件式

サブクエリで条件式を構築するために、`in_subquery`または`not_in_subquery`メソッドを使用する。

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

トランザクションは、ACIDを保証しながら実行するSQL文のグループである。
2つのトランザクションAPIがある。

### `クロージャー`の内部

トランザクションは、クロージャーが`Ok`を返却したときコミットされ、`Err`を返却したときロールバックされる。
2番めと3番めの型パラメーターはそれぞれ`Ok`と`Err`の型である。

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

これは多くのケースで好ましい方法である。
しかしながら、非同期ブロック内で参照のキャプチャを試みている間に、不可能なライフタイムに遭遇する場合は、以下のAPIが答えである。

### `begin` & `commit`/`rollback`

`begin`メソッドはトランザクションを開始して、`commit`または`rollback`メソッドがそれに続く。
もし`txn`がスコープを外れた時、トランザクションは自動的にロールバックされる。

## ストリーミング

効率を改善することを目的に、メモリ割り当てを減らすためには、何らかの`Select`で非同期ストリームを使用する。

```rust
// Stream all fruits
let mut stream = Fruit::find().stream(db).await?;

while let Some(item) = stream.try_next().await? {
    let item: fruit::ActiveModel = item.into();
    // do something with item
}
```

```rust
// Stream all fruits with name contains character "a"
let mut stream = Fruit::find()
    .filter(fruit::Column::Name.contains("a"))
    .order_by_asc(fruit::Column::Name)
    .stream(db)
    .await?;
```

ストリームオブジェクトはドロップされるまで排他的に接続を保持するため、接続が他から借用されることを防ぐ必要があることに注意すること。

```rust
{
    let s1 = Fruit::find().stream(db).await?;
    let s2 = Fruit::find().stream(db).await?;
    let s3 = Fruit::find().stream(db).await?;
    // 3 connections are held
}
// All streams are dropped and connections are returned to the connection pool
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


