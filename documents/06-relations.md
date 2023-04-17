# 関連

- [関連](#関連)
  - [1対1](#1対1)
    - [関連の定義](#関連の定義)
    - [逆の関連の定義](#逆の関連の定義)
  - [1対多](#1対多)
    - [関連の定義](#関連の定義-1)
    - [逆関連の定義](#逆関連の定義)
  - [多対多](#多対多)
    - [関連の定義](#関連の定義-2)
    - [逆関連の定義](#逆関連の定義-1)
  - [連鎖した関連](#連鎖した関連)
    - [遅延ロード](#遅延ロード)
    - [貪欲なロード(Eager Loading)](#貪欲なロードeager-loading)
    - [自己参照](#自己参照)
  - [Bakery Schema](#bakery-schema)

## 1対1

1対1の関連は、最も基本的なデータベースの関連です。
`Cake`エンティティは最大1つの`Fruit`トッピングを持つとします。

### 関連の定義

`Cake`エンティティで、関連を定義します。

1. `Relation`列挙体に新しい`Fruit`バリアントを追加
2. `Fruit`バリアントに`Entity::has_one()`を定義
3. `Related<Entity>`トレイトを実装

```rust
// entity/cake.rs
#[derive(Copy, Clone, Debug, EnumIter)]
pub enum Relation {
    Fruit,
}

impl RelationTrait for Relation {
    fn def(&self) -> RelationDef {
        match self {
            Self::Fruit => Entity::has_one(super::fruit::Entity).into(),
        }
    }
}

impl Related<super::fruit::Entity> for Entity {
    fn to() -> RelationDef {
        Relation::Fruit.def()
    }
}
```

あるいは、その定義は`DeriveRelation`マクロで短縮でき、次は上記の`RelationTrait`に必要な実装を削除します。

```rust
#[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
pub enum Relation {
    #[sea_orm(has_one = "super::fruit::Entity")]
    Fruit,
}

// `Related`トレイトは手動で実装する必要があります
impl Related<super::fruit::Entity> for Entity {
    fn to() -> RelationDef {
        Relation::Fruit.def()
    }
}
```

### 逆の関連の定義

`Fruit`エンティティで、その`cake_id`属性は、`Cake`エンティティのプライマリキーを参照しています。

次の通り逆の関連を定義します。

1. `Fruit`エンティティに、新しい列挙型のバリアント`Relation::Cake`を追加
2. `Entity::belongs_to`メソッドでその定義を記述して、このメソッドを使用して逆関連を定義
3. `Related<cake::Entity>`トレイトを実装

```rust
// entity/fruit.rs
#[derive(Copy, Clone, Debug, EnumIter)]
pub enum Relation {
    Cake,
}

impl RelationTrait for Relation {
    fn def(&self) -> RelationDef {
        match self {
            Self::Cake => Entity::belongs_to(super::cake::Entity)
                .from(Column::CakeId)
                .to(super::cake::Column::Id)
                .into(),
        }
    }
}

impl Related<super::cake::Entity> for Entity {
    fn to() -> RelationDef {
        Relation::Cake.def()
    }
}
```

あるいは、その定義は`DeriveRelation`マクロによって短縮でき、次は`RelationTrait`に必要な実装を削除します。

```rust
#[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
pub enum Relation {
    #[sea_orm(
        belongs_to = "super::cake::Entity",
        from = "Column::CakeId",
        to = "super::cake::Column::Id"
    )]
    Cake,
}

// `Related`トレイトは手動で実装する必要があります。
impl Related<super::cake::Entity> for Entity {
    fn to() -> RelationDef {
        Relation::Cake.def()
    }
}
```

## 1対多

1対多関連は、1対1関連と似ています。
前節で"`Cake`エンティティは最大1つの`Fruit`のトッピングを持つ"と例をあげました。
それを1対多関連にするために、"最大1つ"という制約を削除します。
よって、多くの`Fruit`エンティティを持つかもしれない`Cake`エンティティを持ちます。

### 関連の定義

これは、ほとんど1対多関連の定義と同じで、唯一の違いは、ここで`Entity::has_many()`メソッドを使用することです。

```rust
// entity/cake.rs
#[derive(Copy, Clone, Debug, EnumIter)]
pub enum Relation {
    Fruit,
}

impl RelationTrait for Relation {
    fn def(&self) -> RelationDef {
        match self {
            Self::Fruit => Entity::has_many(super::fruit::Entity).into(),
        }
    }
}

impl Related<super::fruit::Entity> for Entity {
    fn to() -> RelationDef {
        Relation::Fruit.def()
    }
}
```

あるいは、その定義は`DeriveRelation`マクロで短縮することができ、次は、上記の`RelationTrait`に必要な実装を削除します。

```rust
// entity/cake.rs
#[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
pub enum Relation {
    #[sea_orm(has_many = "super::fruit::Entity")]
    Fruit,
}

// `Related`トレイトは手動で実装する必要があります。
impl Related<super::fruit::Entity> for Entity {
    fn to() -> RelationDef {
        Relation::Fruit.def()
    }
}
```

### 逆関連の定義

1対多の逆関連の定義は、1対1の逆関連の定義と同じです。

```rust
// entity/fruit.rs
#[derive(Copy, Clone, Debug, EnumIter)]
pub enum Relation {
    Cake,
}

impl RelationTrait for Relation {
    fn def(&self) -> RelationDef {
        match self {
            Self::Cake => Entity::belongs_to(super::cake::Entity)
                .from(Column::CakeId)
                .to(super::cake::Column::Id)
                .into(),
        }
    }
}

impl Related<super::cake::Entity> for Entity {
    fn to() -> RelationDef {
        Relation::Cake.def()
    }
}
```

あるいは、その定義は`DeriveRelation`マクロで短縮することができ、次は、上記の`RelationTrait`に必要な実装を削除します。

```rust
#[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
pub enum Relation {
    #[sea_orm(
        belongs_to = "super::cake::Entity",
        from = "Column::CakeId",
        to = "super::cake::Column::Id"
    )]
    Cake,
}

// `Related`トレイトは手動で実装する必要があります。
impl Related<super::cake::Entity> for Entity {
    fn to() -> RelationDef {
        Relation::Cake.def()
    }
}
```

## 多対多

多対多関連は3つのテーブルで形成され、2つのテーブルはジャンクションテーブルを介して関連されています。
例として、`Cake`は多くの`Filling`を持ち、`Filling`は中間のエンティティである`CakeFilling`を介して、多くの`Cake`に共有されます。

### 関連の定義

`Cake`エンティティに`Related<filling::Entity>`トレイトを実装します。

SearORMにおける`Relation`は矢です。
その矢は`from`と`to`を持ちます。
`cake_filling::Relation::Cake`は、`CakeFilling -> Cake`を定義します。
[rev](https://docs.rs/sea-orm/*/sea_orm/entity/prelude/struct.RelationDef.html#method.rev)の呼び出しは、それを`Cake -> CakeFilling`に反転します。

`CakeFilling -> Filling`を定義する`cake_filling::Relation::Filling`でこれを連鎖させることは、`Cake -> CakeFilling -> Filling`になります。

```rust
// entity/cake.rs
impl Related<super::filling::Entity> for Entity {
    // 最終的な関連は、`Cake -> CakeFilling -> Filling`です。
    fn to() -> RelationDef {
        super::cake_filling::Relation::Filling.def()
    }

    fn via() -> Option<RelationDef> {
        // オリジナルの関連は、`CakeFilling -> Cake`で、
        // 後の`rev`によって、その関連は`Cake -> CakeFilling`になる。
        Some(super::cake_filling::Relation::Cake.def().rev())
    }
}
```

同様に、`Filling`エンティティで、`Related<cake::Entity>`トレイトを実装します。
まず、`cake_filling::Relation::Filling`逆関連を`介して`中間テーブルと結合して、`cake_filling::Relation::Cake`関連で`Cake`エンティティ`に`結合します。

```rust
// entity/filling.rs
impl Related<super::cake::Entity> for Entity {
    fn to() -> RelationDef {
        super::cake_filling::Relation::Cake.def()
    }

    fn via() -> Option<RelationDef> {
        Some(super::cake_filling::Relation::Filling.def().rev())
    }
}
```

### 逆関連の定義

`CakeFilling`エンティティの`cake_id`属性は`Cake`エンティティのプライマリーキーを参照しており、`filling_id`属性は`Filling`エンティティのプライマリーキーを参照している。

多対多の逆の関連を定義する。

1. `Relation`列挙型に2つの新しいバリアントを追加する。
2. `Entity::belongs_to()`メソッドで2つの関連を定義する。

```rust
#[derive(Copy, Clone, Debug, EnumIter)]
pub enum Relation {
    Cake,
    Filling,
}

impl RelationTrait for Relation {
    fn def(&self) -> RelationDef {
        match self {
            Self::Cake => Entity::belongs_to(super::cake::Entity)
                .from(Column::CakeId)
                .to(super::cake::Column::Id)
                .into(),
            Self::Filling => Entity::belongs_to(super::filling::Entity)
                .from(Column::FillingId)
                .to(super::filling::Column::Id)
                .into(),
        }
    }
}
```

あるいは、定義は`DeriveRelation`マクロで短縮でき、以下は上記と同義である。

```rust
#[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
pub enum Relation {
    #[sea_orm(
        belongs_to = "super::cake::Entity",
        from = "Column::CakeId",
        to = "super::cake::Column::Id"
    )]
    Cake,
    #[sea_orm(
        belongs_to = "super::filling::Entity",
        from = "Column::FillingId",
        to = "super::filling::Column::Id"
    )]
    Filling,
}
```

## 連鎖した関連

2つのエンティティに複数の結合方法がある場合は、複数のエンティティをつなげた複雑な結合がある場合、それを[Linked](https://docs.rs/sea-orm/0.5/sea_orm/entity/trait.Linked.html)で定義できる。
これを`Cake`テーブルと`Filling`テーブルを中間の`CakeFilling`テーブルで結合する、簡単な[例](https://github.com/SeaQL/sea-orm/blob/master/src/tests_cfg/cake.rs)で取り上げる。

```rust
#[derive(Debug)]
pub struct CakeToFilling;

impl Linked for CakeToFilling {
    type FromEntity = cake::Entity;

    type ToEntity = filling::Entity;

    fn link(&self) -> Vec<RelationDef> {
        vec![
            cake_filling::Relation::Cake.def().rev(),
            cake_filling::Relation::Filling.def(),
        ]
    }
}
```

あるいは、`RelationDef`はその場で連鎖した関連を定義でき、以下は上記と同義である。

```rust
#[derive(Debug)]
pub struct CakeToFilling;

impl Linked for CakeToFilling {
    type FromEntity = cake::Entity;

    type ToEntity = filling::Entity;

    fn link(&self) -> Vec<RelationDef> {
        vec![
            cake_filling::Relation::Cake.def().rev(),
            cake_filling::Entity::belongs_to(filling::Entity)
                .from(cake_filling::Column::FillingId)
                .to(filling::Column::Id)
                .into(),
        ]
    }
}
```

### 遅延ロード

[find_linked](https://docs.rs/sea-orm/0.5/sea_orm/entity/prelude/trait.ModelTrait.html#method.find_linked)メソッドで、検索した`Filling`を`Cake`に詰め込むことができる。

```rust
let cake_model = cake::Model {
    id: 12,
    name: "".to_owned(),
};

assert_eq!(
    cake_model
        .find_linked(cake::CakeToFilling)
        .build(DbBackend::MySql)
        .to_string(),
    [
        r#"SELECT `filling`.`id`, `filling`.`name`"#,
        r#"FROM `filling`"#,
        r#"INNER JOIN `cake_filling` ON `cake_filling`.`filling_id` = `filling`.`id`"#,
        r#"INNER JOIN `cake` ON `cake`.`id` = `cake_filling`.`cake_id`"#,
        r#"WHERE `cake`.`id` = 12"#,
    ]
    .join(" ")
);
```

### 貪欲なロード(Eager Loading)

単独の選択で[find_also_linked](https://docs.rs/sea-orm/0.5/sea_orm/entity/prelude/struct.Select.html#method.find_also_linked)メソッドを使用して、すべての`Cake`と`Filling`のペアを検索する。

```rust
assert_eq!(
    cake::Entity::find()
        .find_also_linked(cake::CakeToFilling)
        .build(DbBackend::MySql)
        .to_string(),
    [
        "SELECT `cake`.`id` AS `A_id`, `cake`.`name` AS `A_name`,",
        "`filling`.`id` AS `B_id`, `filling`.`name` AS `B_name`",
        "FROM `cake`",
        "LEFT JOIN `cake_filling` ON `cake`.`id` = `cake_filling`.`cake_id`",
        "LEFT JOIN `filling` ON `cake_filling`.`filling_id` = `filling`.`id`",
    ]
    .join(" ")
);
```

### 自己参照

前のセクションにおいて、[Linked](https://docs.rs/sea-orm/0.5/sea_orm/entity/trait.Linked.html)トレイトを紹介した。
`Linked`トレイトは自己参照する関連を定義できる。

以下の例は自己参照するエンティティを定義する。

```rust
use sea_orm::entity::prelude::*;

#[derive(Clone, Debug, PartialEq, DeriveEntityModel)]
#[sea_orm(table_name = "self_join")]
pub struct Model {
    #[sea_orm(primary_key, auto_increment = false)]
    pub uuid: Uuid,
    pub uuid_ref: Option<Uuid>,
    pub time: Option<Time>,
}

#[derive(Copy, Clone, Debug, EnumIter, DeriveRelation)]
pub enum Relation {
    #[sea_orm(belongs_to = "Entity", from = "Column::UuidRef", to = "Column::Uuid")]
    SelfReferencing,
}

pub struct SelfReferencingLink;

impl Linked for SelfReferencingLink {
    type FromEntity = Entity;

    type ToEntity = Entity;

    fn link(&self) -> Vec<RelationDef> {
        vec![Relation::SelfReferencing.def()]
    }
}
```

## Bakery Schema

![Bakery Schema](https://raw.githubusercontent.com/SeaQL/sea-orm/master/tests/common/bakery_chain/bakery_chain_erd.svg)

さまざまなデータ型と関係を持つ完全なスキーマの例については、SeaORMのテストスイートの[Bakery Schema]で参照できる。
