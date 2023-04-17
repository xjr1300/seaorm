# CRUDの基本

## スキーマの基本

説明のために[基本的なスキーマ](https://github.com/SeaQL/sea-orm/tree/master/src/tests_cfg)を使用します。

- `cake`は`fruit`と1対多です。
- `cake`は`filling`と多対多です。
- `cake_filling`は、`cake`と`filling`の間のジャンクションテーブルです。

![スキーマの基本](https://raw.githubusercontent.com/SeaQL/sea-orm/master/src/tests_cfg/basic_schema.svg)

## 選択 (SELECT)

エンティティを定義したら、データベースからデータを取り出す準備が整います。
データベース内のデータのそれぞれの行は、`Model`に対応します。

デフォルトで、SeaORMは`Column`列挙体に定義されたすべての列を選択します。

### プライマリーキーによる検索

モデルの1つのまたは複合キーで、モデルを検索できます。
選択クエリと条件を自動で構築する[Entity](https://docs.rs/sea-orm/*/sea_orm/entity/trait.EntityTrait.html)の[find_by_id](https://docs.rs/sea-orm/*/sea_orm/entity/trait.EntityTrait.html#method.find_by_id)を呼び出すことから始めます。
そして、`one`メソッドでデータベースから1つのモデルを取得します。

```rust
use super::cake::Entity as Cake;
use super::cake_filling::Entity as CakeFilling;

// プライマリーキーで検索
let cheese: Option<cake::Model> = Cake::find_by_id(1).one(db).await?;
// 複合プライマリーキーで検索
let vanilla: Option<cake_filling::Model> = CakeFilling::find_by_id((6, 8)).one(db).await?;
```

### 条件と順番を伴って検索

プライマリーキーでモデルを取得することに加えて、特定の順番で、特定の条件にマッチした1つまたは複数のモデルも取得できます。
[find](https://docs.rs/sea-orm/*/sea_orm/entity/trait.EntityTrait.html#method.find)メソッドは、SeaORMのクエリビルダーにアクセスできるようにします。
それは`where`と`order by`のようなすべての一般的な選択表現の構築を支援します。
それらは、それぞれ[filter](https://docs.rs/sea-orm/*/sea_orm/entity/prelude/trait.QueryFilter.html#method.filter)と[order_by_*](https://docs.rs/sea-orm/*/sea_orm/query/trait.QueryOrder.html#method.order_by)メソッドを使用することで構築できます。

> 詳細は[条件式](https://www.sea-ql.org/SeaORM/docs/advanced-query/conditional-expression/)を参照してください。

```rust
let chocolate: Vec<cake::Model> = Cake::find()
    .filter(cake::Column::Name.contains("chocolate"))
    .order_by_asc(cake::Column::Name)
    .all(db)
    .await?;
```

### 関連したモデルの検索

> 詳細は[リレーション](https://www.sea-ql.org/SeaORM/docs/relation/one-to-one/)の章を参照してください。

#### 遅延読み込み

[find_related](https://docs.rs/sea-orm/*/sea_orm/entity/prelude/trait.ModelTrait.html#method.find_related)メソッドを使用します。

関連したモデルは、それらを求めたときに必要に応じてロードされるため、何らかのアプリケーションロジックに基づいて関連したモデルを読み込む場合に適しています。
遅延読み込みは、[貪欲な読み込み（Eager Loading）](#貪欲な読み込みeager-loading)と比較して、データベースとのやりとりがが増加することに注意してください。

```rust
// ケーキモデルを最初に検索
let cheese: Option<cake::Model> = Cake::find_by_id(1).one(db).await?;
let cheese: cake::Model = cheese.unwrap();

// そして、このケーキの関連したすべてのフルーツを検索
let fruits: Vec<fruit::Model> = cheese.find_related(Fruit).all(db).await?;
```

#### 貪欲な読み込み(Eager Loading)

すべての関連するモデルが1度で読み込まれます。
これは、遅延読み込みと比較してデータベースとのやりとりを最小化します。

##### 1対1

[find_also_related](https://docs.rs/sea-orm/*/sea_orm/entity/prelude/struct.Select.html#method.find_also_related)メソッドを使用します。

```rust
let fruits_and_cakes: Vec<(fruit::Model, Option<cake::Model>)> = Fruit::find().find_also_related(Cake).all(db).await?;
```

##### 1対多／多対多

[find_with_related](https://docs.rs/sea-orm/*/sea_orm/entity/prelude/struct.Select.html#method.find_with_related)メソッドを使用すると、関連したモデルが親モデルによってグループ化されます。
これを獲得するために、クエリはケーキのプライマリーキーで並び替えられます。
このメソッドは、1対NとM対Nのケース両方を処理して、ジャンクションテーブルががあったとき、関係した2つを結合を実行します。

```rust
let cake_with_fruits: Vec<(cake::Model, Vec<fruit::Model>)> = Cake::find()
    .find_with_related(Fruit)
    .all(db)
    .await?;
```

#### バッチ読み込み

（バージョン）0.11まで、バッチで関連したエンティティを読み込むために[LoaderTrait](https://docs.rs/sea-orm/*/sea_orm/query/trait.LoaderTrait.html)を導入しました。

貪欲な読み込みと比較して、1つ（多対多のケースにおいて、または2つ）以上のデータベースとのやり取りで、帯域幅（1対多のケースを考えると、ある側の行が重複しているかもしれません）を節約します。

##### 1対1

[load_one](https://docs.rs/sea-orm/*/sea_orm/query/trait.LoaderTrait.html#tymethod.load_one)メソッドを使用します。

```rust
let fruits: Vec<fruit::Model> = Fruit::find().all(db).await?;
let cakes: Vec<Option<cake::Model>> = fruits.load_one(Cake, db).await?;
```

##### 1対多

[load_many](https://docs.rs/sea-orm/*/sea_orm/query/trait.LoaderTrait.html#tymethod.load_many)メソッドを使用します。

```rust
let cakes: Vec<cake::Model> = Cake::find().all(db).await?;
let fruits: Vec<Vec<fruit::Model>> = cakes.load_many(Fruit, db).await?;
```

##### 多対多

[load_many_to_many](https://docs.rs/sea-orm/*/sea_orm/query/trait.LoaderTrait.html#tymethod.load_many_to_many)メソッドを使用します。
ジャンクションエンティティを提供する必要があります。

```rust
let cakes: Vec<cake::Model> = Cake::find().all(db).await?;
let fillings: Vec<Vec<filling::Model>> = cakes.load_many_to_many(Filling, CakeFilling, db).await?;
```

### 結果のページネート

カスタムなページサイズで任意のSeaORMの選択を[paginator](https://docs.rs/sea-orm/*/sea_orm/struct.Paginator.html)に変換します。

```rust
use sea_orm::{entity::*, query::*, tests_cfg::cake};
let mut cake_pages = cake::Entity::find()
    .order_by_asc(cake::Column::Id)
    .paginate(db, 50);

while let Some(cakes) = cake_pages.fetch_and_next().await? {
    // Vec<cake::Model>型のcakesで何かする
}
```

### カーソルページネーション

プライマリーキーのような列（複数列）に基づいて行をページネートするために、カーソルページネーションを使用します。

```rust
use sea_orm::{entity::*, query::*, tests_cfg::cake};
// `cake`.`id`で並び替えられたカーソルを作成
let mut cursor = cake::Entity::find().cursor_by(cake::Column::Id);

// `cake`.`id` > 1 AND `cake`.`id` < 100でページネートした結果をフィルタ
cursor.after(1).before(100);

// （`cake`.`id`を昇順で並び替えした）最初の10行を取得
for cake in cursor.first(10).all(db).await? {
    // cake::Model型のcakeで何かする
}

// （`cake`.`id`の降順で並び替えるが、行は昇順で返却されている）最後の10行を取得
for cake in cursor.last(10).all(db).await? {
    // cake::Model型のcakeで何かする
}
```

複合プライマリーキーに基づいて行をページねーとすることもできます。

```rust
use sea_orm::{entity::*, query::*, tests_cfg::cake_filling};
let rows = cake_filling::Entity::find()
    .cursor_by((cake_filling::Column::CakeId, cake_filling::Column::FillingId))
    .after((0, 1))
    .before((10, 11))
    .first(3)
    .all(&db)
    .await?,
```

### カスタムな選択

カスタム列や条件で選択したい場合は、[カスタムな選択](https://www.sea-ql.org/SeaORM/docs/advanced-query/custom-select/)を参照してください。

## 挿入

SeaORMの挿入に深入りする前に、`ActiveValue`と`ActiveModel`を説明する必要があります。

### ActiveValue

`ActiveModel`属性にされた変更をキャプチャーするためのラッパー構造体です。

```rust
use sea_orm::ActiveValue::{Set, NotSet, Unchanged};

// 設定した値
let _: ActiveValue<i32> = Set(10);

// NotSet値
let _: ActiveValue<i32> = NotSet;

// `変更されていない`ことを示す値
let v: ActiveValue<i32> = Unchanged(10);

// `変更されていない`ことを示す現在の値を`Set`した値に変換
assert!(v.reset(), Set(10));
```

### ModelとActiveModel

`ActiveModel`は`ActiveValue`にラップされた`Model`のすべての属性を持ちます。

列集合の部分集合で行を挿入するために、`ActiveModel`を使用できます。

```rust
let cheese: Option<cake::Model> = Cake::find_by_id(1).one(db).await?;

// モデルを取得
let model: cake::Model = cheese.unwrap();
assert_eq!(model.name, "Cheese Cake".to_owned());

// ActiveModelに入れる
let active_model: cake::ActiveModel = model.into();
assert_eq!(active_model.name, ActiveValue::unchanged("Cheese Cake".to_owned()));
```

#### JSON値からActiveModelを設定

もし、ユーザー入力をデータベースに保存したい場合、簡単にJSON値を`ActiveModel`変換できます。
JSONのプライマリーキーの[デシリアライズをスキップ](https://serde.rs/attr-skip-serializing.html)したい場合は、次の通りそれを設定できることに注意してください。

```rust
#[derive(Clone, Debug, PartialEq, Eq, DeriveEntityModel, Serialize, Deserialize)]
#[sea_orm(table_name = "fruit")]
pub struct Model {
    #[sea_orm(primary_key)]
    #[serde(skip_deserializing)] // デシリアライズをスキップ
    pub id: i32,
    pub name: String,
    pub cake_id: Option<i32>,
}
```

`set_from_json`メソッドで`ActiveModel`の属性を設定します。

```rust
// プライマリーキーが設定されたActiveModel
let mut fruit = fruit::ActiveModel {
    id: ActiveValue::Set(1),
    name: ActiveValue::NotSet,
    cake_id: ActiveValue::NotSet,
};

// このメソッドはActiveModelのプライマリーキーの値を変更しないことに注意
fruit.set_from_json(json!({
    "id": 8,
    "name": "Apple",
    "cake_id": 1,
}))?;

assert_eq!(
    fruit,
    fruit::ActiveModel {
        id: ActiveValue::Set(1),    // 上で8を設定したにも関わらず、最初に設定した1が設定されている
        name: ActiveValue::Set("Apple".to_owned()),
        cake_id: ActiveValue::Set(Some(1)),
    }
);
```

`from_json`メソッドで、JSON値から新しい`ActiveModel`を作成します。

```rust
let fruit = fruit::ActiveModel::from_json(json!({
    "name": "Apple",
}))?;

assert_eq!(
    fruit,
    fruit::ActiveModel {
        id: ActiveValue::NotSet,
        name: ActiveValue::Set("Apple".to_owned()),
        cake_id: ActiveValue::NotSet,
    }
);
```

### ActiveModelが変更されているか確認する

[is_changed](https://docs.rs/sea-orm/*/sea_orm/entity/prelude/trait.ActiveModelTrait.html#method.is_changed)メソッドで`ActiveModel`の任意のフィールドが`Set`されているか確認できます。

```rust
let mut fruit: fruit::ActiveModel = Default::default();
assert!(!fruit.is_changed());

fruit.set(fruit::Column::Name, "apple".into());
assert!(fruit.is_changed());
```

### ActiveModelをModelに変換する（戻す）

[try_into_model](https://docs.rs/sea-orm/*/sea_orm/entity/trait.TryIntoModel.html#tymethod.try_into_model)メソッドを使用して、`ActiveModel`から`Model`に戻せます。

```rust
assert_eq!(
    ActiveModel {
        id: Set(2),
        name: Set("Apple".to_owned()),
        cake_id: Set(Some(1)),
    }
    .try_into_model()
    .unwrap(),
    Model {
        id: 2,
        name: "Apple".to_owned(),
        cake_id: Some(1),
    }
);

assert_eq!(
    ActiveModel {
        id: Set(1),
        name: NotSet,
        cake_id: Set(None),
    }
    .try_into_model(),
    Err(DbErr::AttrNotSet(String::from("name")))
);
```

### 1行挿入する

`ActiveModel`を挿入して、新たな`Model`を取り戻します。
データベースから返却されたその値は、任意の自動生成されたフィールドはすべて入力されています。

```rust
let pear = fruit::ActiveModel {
    name: Set("Pear".to_owned()),
    ..Default::default() // 他のすべての値は`NotSet`
};

let pear: fruit::Model = pear.insert(db).await?;
```

`ActiveModel`を挿入して、最後に挿入したIDを取得します。
その型はモデルのプライマリーキーの型にマッチするため、もしモデルが複合プライマリーキーを持つ場合、タプルになります。

```rust
let pear = fruit::ActiveModel {
    name: Set("Pear".to_owned()),
    ..Default::default() // all other attributes are `NotSet`
};

let res: InsertResult = fruit::Entity::insert(pear).exec(db).await?;
assert_eq!(res.last_insert_id, 28)
```

### 多くの行の挿入する

多くの`ActiveModel`を挿入して、最後に挿入されたIDを取得します。

```rust
let apple = fruit::ActiveModel {
    name: Set("Apple".to_owned()),
    ..Default::default()
};

let orange = fruit::ActiveModel {
    name: Set("Orange".to_owned()),
    ..Default::default()
};

let res: InsertResult = Fruit::insert_many([apple, orange]).exec(db).await?;
assert_eq!(res.last_insert_id, 30)
```

### 競合

競合する振る舞いを起こす`ActiveMode`を挿入します。

```rust
let orange = cake::ActiveModel {
    id: ActiveValue::set(2),
    name: ActiveValue::set("Orange".to_owned()),
};

assert_eq!(
    cake::Entity::insert(orange.clone())
        .on_conflict(
            // 競合で何もしない
            sea_query::OnConflict::column(cake::Column::Name)
                .do_nothing()
                .to_owned()
        )
        .build(DbBackend::Postgres)
        .to_string(),
    r#"INSERT INTO "cake" ("id", "name") VALUES (2, 'Orange') ON CONFLICT ("name") DO NOTHING"#,
);

assert_eq!(
    cake::Entity::insert(orange)
        .on_conflict(
            // 競合で更新
            sea_query::OnConflict::column(cake::Column::Name)
                .update_column(cake::Column::Name)
                .to_owned()
        )
        .build(DbBackend::Postgres)
        .to_string(),
    r#"INSERT INTO "cake" ("id", "name") VALUES (2, 'Orange') ON CONFLICT ("name") DO UPDATE SET "name" = "excluded"."name""#,
);
```

挿入したり、任意の行を更新することのないUPSERT文を実行は、`DbErr::RecordNotInserted`エラーの結果になります。

```rust
// `id`列が競合する値のとき、何もしない
let on_conflict = OnConflict::column(Column::Id).do_nothing().to_owned();

// `1`、`2`、`3`をテーブルに挿入
let res = Entity::insert_many([
    ActiveModel { id: Set(1) },
    ActiveModel { id: Set(2) },
    ActiveModel { id: Set(3) },
])
.on_conflict(on_conflict.clone())
.exec(db)
.await;

assert_eq!(res?.last_insert_id, 3);

// 前の3つの行と一緒に、`4`をテーブルに挿入
let res = Entity::insert_many([
    ActiveModel { id: Set(1) },
    ActiveModel { id: Set(2) },
    ActiveModel { id: Set(3) },
    ActiveModel { id: Set(4) },
])
.on_conflict(on_conflict.clone())
.exec(db)
.await;

assert_eq!(res?.last_insert_id, 4);

// 最後の挿入を報告します。
// すべて4つの行が既に存在されているため、これは本質的に何もしません。
// `DbErr::RecordNotInserted`エラーがスローされるでしょう。
let res = Entity::insert_many([
    ActiveModel { id: Set(1) },
    ActiveModel { id: Set(2) },
    ActiveModel { id: Set(3) },
    ActiveModel { id: Set(4) },
])
.on_conflict(on_conflict)
.exec(db)
.await;

assert_eq!(res.err(), Some(DbErr::RecordNotInserted));
```

## 更新

### 1行更新する

検索した結果から`Model`を取得します。
もし、`Model`をデータベースに戻して保存する場合、最初に`Model`を`ActiveModel`に変換する必要があります。
生成されたクエリは`Set`属性のみが含まれます。

```rust
let pear: Option<fruit::Model> = Fruit::find_by_id(28).one(db).await?;

// ActiveModelに変換
let mut pear: fruit::ActiveModel = pear.unwrap().into();

// name属性を更新
pear.name = Set("Sweet pear".to_owned());

// SQL: `UPDATE "fruit" SET "name" = 'Sweet pear' WHERE "id" = 28`
let pear: fruit::Model = pear.update(db).await?;
```

すべての属性を更新するために、`Unchanged`を`Set`に変換します。

```rust
// ActiveModelに入れる
let mut pear: fruit::ActiveModel = pear.into();

// name属性を更新
pear.name = Set("Sweet pear".to_owned());

// "dirty"として特定の属性を設定（強制的に更新）
pear.reset(fruit::Column::CakeId);
// または、"dirty"としてすべての属性を設定（強制的に更新）
pear.reset_all();

// SQL: `UPDATE "fruit" SET "name" = 'Sweet pear', "cake_id" = 10 WHERE "id" = 28`
let pear: fruit::Model = pear.update(db).await?;
```

### 複数行更新する

更新したいそれぞれの`Model`を検索しないで、SeaORMの選択によって、データベース内の複数の行を更新できます。

SeaORMの選択してそれぞれの`Model`を検索しないで、データベース内の複数行を更新することもできます。

```rust
// ActiveModelを使用して、属性を一括で設定
let update_result: UpdateResult = Fruit::update_many()
    .set(pear)
    .filter(fruit::Column::Id.eq(1))
    .exec(db)
    .await?;

// UPDATE `fruit` SET `cake_id` = 1 WHERE `fruit`.`name` LIKE '%Apple%'
Fruit::update_many()
    .col_expr(fruit::Column::CakeId, Expr::value(1))
    .filter(fruit::Column::Name.contains("Apple"))
    .exec(db)
    .await?;
```

## 保存

これ（`save`メソッド）は、（挿入／更新した）アクティブモデルをデータベースに保存するためのヘルパーメソッドです。

### 保存の振る舞い

`ActiveModel`を保存するとき、プライマリーキー属性に依存して、挿入または更新のとちらかが実行されます。

- プライマリーキーが`NotSet`の場合は挿入
- プライマリーキーが`Set`または`Unchanged`の場合は更新

### 使用方法

`ActiveModel`を挿入または更新するために`save`メソッドを呼び出します。

```rust
use sea_orm::ActiveValue::NotSet;

let banana = fruit::ActiveModel {
    id: NotSet, // プライマリーキーは`NotSet`
    name: Set("Banana".to_owned()),
    ..Default::default() // 他のすべての属性は`NotSet`
};

// プライマリーキーの`id`が`NotSet`であるため挿入
let banana: fruit::ActiveModel = banana.save(db).await?;

banana.name = Set("Banana Mongo".to_owned());

// プライマリーキーの`id`が`Unchanged`であるため更新
let banana: fruit::ActiveModel = banana.save(db).await?;
```

## 削除

### 1行削除する

データベースから`Model`を検索して、対応する行をデータベースから削除します。

```rust
use sea_orm::entity::ModelTrait;

let orange: Option<fruit::Model> = Fruit::find_by_id(30).one(db).await?;
let orange: fruit::Model = orange.unwrap();

let res: DeleteResult = orange.delete(db).await?;
assert_eq!(res.rows_affected, 1);
```

### プライマリーキーで削除する

データベースから`Model`を選択してからそれを削除する代わりに、そのプライマリーキーによってデータベースから直接行を削除することもできます。

```rust
let res: DeleteResult = Fruit::delete_by_id(38).exec(db).await?;
assert_eq!(res.rows_affected, 1);
```

### 複数行削除する

SeaORMの選択でそれぞれの`Model`を検索することなしに、データベースから複数の行を削除することもできます。

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
