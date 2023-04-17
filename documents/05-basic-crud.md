# CRUDの基本

## スキーマの基本

説明のために[基本的なスキーマ](https://github.com/SeaQL/sea-orm/tree/master/src/tests_cfg)を使用します。

- `cake`は`fruit`と1対多です。
- `cake`は`filling`と多対多です。
- `cake_filling`は、`cake`と`filling`の間のジャンクションテーブルです。

![スキーマの基本](https://raw.githubusercontent.com/SeaQL/sea-orm/master/src/tests_cfg/basic_schema.svg)
## Select

エンティティを定義すれば、データベースからデータを取り出す準備が整う。
データベース内のデータを示すそれぞれの行は`Model`に対応する。

デフォルトでは、SeaORMは`Column`列挙体で定義されたすべての列を選択する。

### プライマリーキーで検索

モデルのプライマリーキーでモデルを検索する。
プライマリキーは、単独のキーまたは複合キーである。
自動的に選択クエリと条件を構築する[Entity](https://docs.rs/sea-orm/0.5/sea_orm/entity/trait.EntityTrait.html)の[find_by_id](https://docs.rs/sea-orm/0.5/sea_orm/entity/trait.EntityTrait.html#method.find_by_id)の呼び出しから開始する。
そして、`one`メソッドでデータベースから単独のモデルを取得する。

```rust
use super::cake::Entity as Cake;
use super::cake_filling::Entity as CakeFilling;

// Find by primary key
let cheese: Option<cake::Model> = Cake::find_by_id(1).one(db).await?;

// Find by composite primary keys
let vanilla: Option<cake_filling::Model> = CakeFilling::find_by_id((6, 8)).one(db).await?;
```

### 条件で検索して並び替える

プライマリーキーでモデルを検索することに加えて、記述した条件に適合する1つまたは複数のモデルを特定の条件で取得できる。
[find](https://docs.rs/sea-orm/0.5/sea_orm/entity/trait.EntityTrait.html#method.find)メソッドは、SeaORMのクエリビルダーにアクセスする機能を提供する。
`find`メソッドは、`where`や`order by`のようなすべての選択式の構築をサポートする。
`where`や`order by`は、それぞれ[filter](https://docs.rs/sea-orm/0.5/sea_orm/entity/prelude/trait.QueryFilter.html#method.filter)と[order_by_*](https://docs.rs/sea-orm/0.5/sea_orm/query/trait.QueryOrder.html#method.order_by)メソッドを使用することで構築される。

:::note info
条件式の詳細は[conditional expression](https://www.sea-ql.org/SeaORM/docs/advanced-query/conditional-expression)を読むこと。
:::

### 関連するモデルの検索

:::note info
テーブルの結合の詳細は[table joins](https://www.sea-ql.org/SeaORM/docs/advanced-query/custom-joins)を読むこと。
:::

#### 遅延ロード

[find_related](https://docs.rs/sea-orm/0.5/sea_orm/entity/prelude/trait.ModelTrait.html#method.find_related)メソッドを使用する。

関連するモデルは、それらが求められたときに必要に応じてロードされるため、何らかのアプリケーションロジックに基づいて関連するモデルをロードする場合に適している。
遅延ロードは`Eager Loading`(貪欲なロード:一括でデータを取り出す)と比較して、データベースとのやりとりがが増加することに注意すること。

```rust
// Find a cake model first
let cheese: Option<cake::Model> = Cake::find_by_id(1).one(db).await?;
let cheese: cake::Model = cheese.unwrap();

// Then, find all related fruits of this cake
let fruits: Vec<fruit::Model> = cheese.find_related(Fruit).all(db).await?;
```

#### 貪欲なロード(Eager Loading)

すべての関連するモデルが1度にロードされる。
これは、遅延ロードと比較してデータベースとのやりとりのオーバーヘッドを最小化する。

##### 1対1

[find_also_related](https://docs.rs/sea-orm/0.5/sea_orm/entity/prelude/struct.Select.html#method.find_also_related)メソッドを使用する。

```rust
let cake_and_fruit: Vec<(cake::Model, Option<fruit::Model>)> = Cake::find().find_also_related(Fruit).all(db).await?;
```

##### 1対多

[find_with_related](https://docs.rs/sea-orm/0.5/sea_orm/entity/prelude/struct.Select.html#method.find_with_related)メソッドを使用すると、関連するモデルは親モデルによってグループ化される。

```rust
let cake_with_fruits: Vec<(cake::Model, Vec<fruit::Model>)> = Cake::find()
    .find_with_related(Fruit)
    .all(db)
    .await?;
```

### 結果のページ付け

カスタムなページサイズでSeaORMの選択結果を[paginator](https://docs.rs/sea-orm/0.5/sea_orm/struct.Paginator.html)に変換する。

```rust
use sea_orm::{entity::*, query::*, tests_cfg::cake};
let mut cake_pages = cake::Entity::find()
    .order_by_asc(cake::Column::Id)
    .paginate(db, 50);

while let Some(cakes) = cake_pages.fetch_and_next().await? {
    // Do something on cakes: Vec<cake::Model>
}
```

### カスタムな選択

カスタム列や条件で選択したい場合は、[custom select](https://www.sea-ql.org/SeaORM/docs/advanced-query/custom-select)を読むこと。

## 挿入

SeaORMの挿入を深掘りする前に、`ActiveValue`と`ActiveModel`を紹介する。

### ActiveValue

`ActiveModel`属性になされた変更を捉えるラッパー構造体である。

```rust
use sea_orm::ActiveValue::NotSet;

// Set value
let _: ActiveValue<i32> = Set(10);

// NotSet value
let _: ActiveValue<i32> = NotSet;
```

### ModelとActiveModel

`ActiveModel`は`ActiveValue`にラップされたモデルのすべての属性を持つ。

ActiveModelを使用して、列のサブセットが設定された行を挿入できる。

```rust
let cheese: Option<cake::Model> = Cake::find_by_id(1).one(db).await?;

// Get Model
let model: cake::Model = cheese.unwrap();
assert_eq!(model.name, "Cheese Cake".to_owned());

// Into ActiveModel
let active_model: cake::ActiveModel = model.into();
assert_eq!(active_model.name, ActiveValue::unchanged("Cheese Cake".to_owned()));
```

### 1行挿入

`ActiveModel`を挿入して最新の`Model`を得る。
`Model`の値はデータベースから取得されており、自動生成フィールドが生成される。

```rust
let pear = fruit::ActiveModel {
    name: Set("Pear".to_owned()),
    ..Default::default() // all other attributes are `NotSet`
};

let pear: fruit::Model = pear.insert(db).await?;
```

`ActiveModel`を挿入して、最後に挿入したIDを取得する。
IDは`Model`のプライマリーキーの型にマッチしており、モデルが複合キーを持つ場合はタプルになる。

```rust
let pear = fruit::ActiveModel {
    name: Set("Pear".to_owned()),
    ..Default::default() // all other attributes are `NotSet`
};

let res: InsertResult = fruit::Entity::insert(pear).exec(db).await?;
assert_eq!(res.last_insert_id, 28)
```

### 多くの行の挿入

多くの`ActiveModel`を挿入して最後に挿入されたIDを取得する。

```rust
let apple = fruit::ActiveModel {
    name: Set("Apple".to_owned()),
    ..Default::default()
};

let orange = fruit::ActiveModel {
    name: Set("Orange".to_owned()),
    ..Default::default()
};

let res: InsertResult = Fruit::insert_many(vec![apple, orange]).exec(db).await?;
assert_eq!(res.last_insert_id, 30)
```

## 更新

### 1行更新

`find`メソッドの結果から`Model`を得られる。
`Model`をデータベースに戻して保存する場合、最初に`Model`を`ActiveModel`に変換する必要がある。
生成されたクエリは`Set`属性のみが含まれる。

```rust
let pear: Option<fruit::Model> = Fruit::find_by_id(28).one(db).await?;

// ActiveModelに変換。
let mut pear: fruit::ActiveModel = pear.unwrap().into();

// name属性を更新。
pear.name = Set("Sweet pear".to_owned());

// プライマリーキーを使用してデータベースの対応する行を更新。
let pear: fruit::Model = pear.update(db).await?;
```

### 複数行の更新

更新したいそれぞれの`Model`を検索しないで、SeaORMの選択によって、データベース内の複数の行を更新できる。

```rust
// Bulk set attributes using ActiveModel
let update_result: UpdateResult = Fruit::update_many()
    .set(pear)
    .filter(fruit::Column::Id.eq(1))
    .exec(db)
    .await?;

// UPDATE `fruit` SET `cake_id` = NULL WHERE `fruit`.`name` LIKE '%Apple%'
Fruit::update_many()
    .col_expr(fruit::Column::CakeId, Expr::value(Value::Null))
    .filter(fruit::Column::Name.contains("Apple"))
    .exec(db)
    .await?;
```

## 保存

`save`メソッドは、データベースに`ActiveModel`を挿入または更新する保存するヘルパーメソッドである。

### 保存の振る舞い

`ActiveModel`を保存するとき、プライマリキー属性によって挿入または更新のどちらかが実行される。

* プライマリーキーが`NotSet`の場合は挿入
* プライマリキーが`Set`または`Unchanged`の場合は更新

### 使用方法

`save`メソッドを呼び出すと`ActiveModel`を挿入または更新する。

```rust
use sea_orm::ActiveValue::NotSet;

let banana = fruit::ActiveModel {
    id: NotSet, // primary key is NotSet
    name: Set("Banana".to_owned()),
    ..Default::default() // all other attributes are `NotSet`
};

// Insert, because primary key `id` is `NotSet`
let banana: fruit::ActiveModel = banana.save(db).await?;

banana.name = Set("Banana Mongo".to_owned());

// Update, because primary key `id` is `Unchanged`
let banana: fruit::ActiveModel = banana.save(db).await?;
```

## 削除

### 1行削除

データベースから`Model`を検索して、対応する行をデータベースから削除する。

```rust
let orange: Option<fruit::Model> = Fruit::find_by_id(30).one(db).await?;
let orange: fruit::Model = orange.unwrap();

let res: DeleteResult = orange.delete(db).await?;
assert_eq!(res.rows_affected, 1);
```

### 複数行削除

`Model`を検索しないで、SeaORMの選択で、データベースから複数の行を削除できる。

```rust
// DELETE FROM `fruit` WHERE `fruit`.`name` LIKE '%Orange%'
let res: DeleteResult = fruit::Entity::delete_many()
    .filter(fruit::Column::Name.contains("Orange"))
    .exec(db)
    .await?;

assert_eq!(res.rows_affected, 2);
```

## JSON

### JSONでシリアライズした結果を選択

すべてのSeaORMの選択は`serde_json::Value`を返却することができる。

```rust
// Find by id
let cake: Option<serde_json::Value> = Cake::find_by_id(1)
    .into_json()
    .one(db)
    .await?;

assert_eq!(
    cake,
    Some(serde_json::json!({
        "id": 1,
        "name": "Cheese Cake"
    }))
);

// Find with filter
let cakes: Vec<serde_json::Value> = Cake::find()
    .filter(cake::Column::Name.contains("chocolate"))
    .order_by_asc(cake::Column::Name)
    .into_json()
    .all(db)
    .await?;

assert_eq!(
    cakes,
    vec![
        serde_json::json!({
            "id": 2,
            "name": "Chocolate Forest"
        }),
        serde_json::json!({
            "id": 8,
            "name": "Chocolate Cupcake"
        }),
    ]
);

// Paginate json result
let cake_pages: Paginator<_> = Cake::find()
    .filter(cake::Column::Name.contains("chocolate"))
    .order_by_asc(cake::Column::Name)
    .into_json()
    .paginate(db, 50);

while let Some(cakes) = cake_pages.fetch_and_next().await? {
    // Do something on cakes: Vec<serde_json::Value>
}
```

## SQL実行

### SQLでクエリする

MySQLとSQLiteは`?`で、PostgreSQLは`$N`で、 パラメータをバインドする適切なパラメーターを使用することで、SQLでモデルを選択できる。

```rust
let cheese: Option<cake::Model> = cake::Entity::find()
    .from_raw_sql(Statement::from_sql_and_values(
        DbBackend::Postgres,
        r#"SELECT "cake"."id", "cake"."name" FROM "cake" WHERE "id" = $1"#,
        vec![1.into()],
    ))
    .one(&db)
    .await?;
```

また、カスタムモデルを選択できる。
ここで、ケーキからすべてのユニークな名前を選択する。

```rust
#[derive(Debug, FromQueryResult)]
pub struct UniqueCake {
    name: String,
}

let unique: Vec<UniqueCake> = UniqueCake::find_by_statement(Statement::from_sql_and_values(
        DbBackend::Postgres,
        r#"SELECT "cake"."name" FROM "cake" GROUP BY "cake"."name"#,
        vec![],
    ))
    .all(&db)
    .await?;
```

### SQLクエリの取得

いずれかのCRUD操作で`build`と`to_string`メソッドを使用して、デバッグ目的でデータベース特有のSQLを取得できる。

```rust
use sea_orm::DatabaseBackend;

assert_eq!(
    cake_filling::Entity::find_by_id((6, 8))
        .build(DatabaseBackend::MySql)
        .to_string(),
    vec![
        "SELECT `cake_filling`.`cake_id`, `cake_filling`.`filling_id` FROM `cake_filling`",
        "WHERE `cake_filling`.`cake_id` = 6 AND `cake_filling`.`filling_id` = 8",
    ].join(" ")
);
```

## SQLクエリの使用と実行インターフェース

`sea-query`を使用してSQL文を生成して、SeaORM内の`DatabaseConnection`インターフェースを介して、SQL文による問い合わせや実行することができる。

### `query_one`と`query_all`メソッドを使用した結果の取得

```rust
let query_res: Option<QueryResult> = db
    .query_one(Statement::from_string(
        DatabaseBackend::MySql,
        "SELECT * FROM `cake`;".to_owned(),
    ))
    .await?;
let query_res = query_res.unwrap();
let id: i32 = query_res.try_get("", "id")?;

let query_res_vec: Vec<QueryResult> = db
    .query_all(Statement::from_string(
        DatabaseBackend::MySql,
        "SELECT * FROM `cake`;".to_owned(),
    ))
    .await?;
```

## `execute`メソッドを使用したクエリの実行

```rust
let exec_res: ExecResult = db
    .execute(Statement::from_string(
        DatabaseBackend::MySql,
        "DROP DATABASE IF EXISTS `sea`;".to_owned(),
    ))
    .await?;
assert_eq!(exec_res.rows_affected(), 1);
```
