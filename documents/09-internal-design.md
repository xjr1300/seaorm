# 内部実装

## トレイトと型

### Entity

[EntityTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)トレイトを実装したユニット構造体は、データベースのテーブルを表現する。

このトレイトは、次のエンティティの属性を含んでいる。

* テーブル名（[EntityName](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)を実装）
* 列（[ColumnTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)を実装）
* 関連（[RelationTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)を実装）
* プライマリーキー（[PrimaryKeyTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)と[PrimaryKeyToColumn](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)を実装）

このトレイトはCRUD操作のためにAPIを提供する。

* 選択: `find`、`find_*`
* 挿入: `insert`、`insert_*`
* 更新: `update`、`update_*`
* 削除: `delete`、`delete_*`

### Column

[ColumnTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)トレイトを実装した列挙型は、テーブルのすべての列と列の型と属性を表現する。

このトレイトは以下を実装する。

* [IdenStatic](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)は静的ライフタイムで列の識別子とマッピングする機能を提供する。
* [Iterable](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)はSeaORMコアにすべての列のバリアントをイテレートさせる。

### Primary Key

[PrimaryKeyTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)トレイトを実装した列挙型でプライマリーキーを表現する。
それぞれのプライマリーキーのバリアントは、列のバリアントとの対応を持つ。

このトレイトは以下を実装する。

* [IdenStatic](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)は静的ライフタイムで列の識別子とマッピングする機能を提供する。
* [Iterable](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)はSeaORMコアにすべての列のバリアントをイテレートさせる。

### Model

[ModelTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)を実装した構造体で、メモリにクエリの結果を蓄積する。
これは、読み込み目的で使用することを意図しており、ステートレスである。

このトレイトは以下を実装する。

* [FromQueryResult](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)はクエリの結果を対応するモデルに変換する。

### Active Model

[ActiveModelTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)を実装した構造体で、挿入/更新操作を表現する。
これは編集されてデータベースに保存されることを意図している。

このトレイトは以下を実装する。

* [ActiveModelBehavior](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)はアクティブモデルの異なる操作のハンドラを定義する。

### Active Enum

[ActiveEnum](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)を実装した列挙型でRUSTの列挙型のバリアントとして、データベースに保存される値を表現する。

### Relation

[RelationTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)を実装した列挙型で他のエンティティとの関連を定義する。

このトレイトは以下を実装する。

* [Iterable](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)は、SeaORMコアにすべての関連のバリアントをイテレートさせるう。

### Related

ジェネリックトリトで、[Related](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)は、関連するエンティティ同士をクエリする結合パスを定義して、特に多対多関連で便利である。

### Linked

[Linked](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)トレイトは、つながった関連、自己関連および2つのエンティティの複数の関連を含む複雑な結合パスを定義する。

## マクロによる派生

### EntityModel

[EntityModel](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)派生マクロは、与えられたモデルから`Entity`、`Column`および`PrimaryKey`を自動的に生成する、'全能な'マクロである。

### Entity

`DeriveEntity`派生マクロは、`Entity`のために[EntityTrait]()を実装して、現在のスコープ内の既存の`Model`、`Column`、`PrimaryKey`および関連を類推する。
また、`Entity`のために[Iden](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)と[IdenStatic](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)の実装を提供する。

### Column

[DeriveColumn](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)派生マクロは、`Column`のために[ColumnTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)を実装する。
[Iden](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)と[IdenStatic](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)を実装することにより、各列の識別子を定義する。
[EnumIter](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)も派生されており、すべての列挙型のバリアントをイテレーションさせる。

### Primary Key

[DerivePrimaryKey](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)派生マクロは`PrimaryKey`のために、プライマリキーと列の間の面倒なマッピングを定義する[PriamryKeyToColumn](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)を実装する。
[EnumIter](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)も派生されており、すべての列挙型のバリアントをイテレーションさせる。

### Model

[DeriveModel](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)派生マクロは、`Model`のためにモデルのすべての属性のセッターとゲッターを提供する`ModelTrait`を実装する。
また、クエリ結果を対応する`Model`に変換する[FromQueryResult](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)を実装する。

### Active Model

[DeriveActiveModel](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)派生マクロは`ActriveModel`のためにアクティブモデル内のすべてのアクティブバリューのセッターとゲッターを提供する[ActiveModelTrait]を実装する。

### Relation

[DeriveRelation](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)派生マクロは`Relation`のために[RelationTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)を実装する。

### Iterable

[EnumIter](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)派生マクロは、列挙型のすべての￥バリアントを￥イテレーションするために[Iterable](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)を実装する。

## Dieselとの比較

[SeaORM](https://github.com/SeaQL/sea-orm)と[Diesel](https://github.com/diesel-rs/diesel)は、データベースの完全なインターフェースを提供するという、同じゴールを共有している。

SeaORMとDieselの両方は、MySQL、PostgreSQLおよびSQLiteで動作するため、どちらかを選択する必要はない。
Dieselはサードパーティが特殊なバックエンドを実装することを許し、SeaORMは許さない。

SeaORMがDieselとは異なる選択をしたことがある。

### アーキテクチャ

最初に、SeaORMはおそらく最も求められた機能であるRustの非同期をサポートしている。
現在、非同期を使用してもパフォーマンスが向上しないことがあるが、非同期プログラミングは、早い段階で行う必要のあるアーキテクチャ上の決定である。
SeaORMの選択により、Rustの非同期エコシステムが成熟するのが待ち遠しくなる。

内部で、SeaORMと[SQLx](https://crates.io/crates/sqlx)は、純粋なRustの技術スタックを提供する。
Dieselはデフォルトでネイティブドライバを使用して、ネイティブドライバを純粋なRustのドライバに置き換えることは、いくらかの努力が必要になる。
どちらにも利点と欠点があり、どちらを使用するかは好みである。

SeaORMはモジュラー設計を採用している。
ORMのアイデアが好きでなくても、基盤となるクエリビルダーのある[SeaORM](https://crates.io/crates/sea-query)を利用することを勧める。
SeaORMは軽量で簡単いプロジェクトに統合できる。
SeaORMを使用する場合はSeaORMのAPIを利用でき、必要であれば柔軟なクエリビルダの能力で高いレベルの抽象化の利点を得られる。

### プログラミングパラダイム

同期と非同期の基盤に加えて、DieselとSeaORMには静的と動的といった大きな違いがある。

Dieselは、型が完全に静的にチェックされるコンパイルタイムAPIを提供する。
Dieselで動的なクエリを実行できるが、コンパイルタイムチェックのいくつかの利点を失うことがある。

SeaORMは動的で、ランタイムで確立される。
SeaORMはより柔軟である。
いくつかのコンパイルタイムチェックを失っても、SeaORMは代わりにテストで正確性を声明することを助ける。

両方のライブ拉致はトレイトやジェネリクスを多用しているが、SeaORMは少ない型を生成する。
Dieselの各列はさまざまなトレイトを実装した構造体で、SeaORMの各列は列挙体のバリアントで、その列挙体はいくつかのトレイトを実装したエンティティのすべての列をまとめている。

SeaORMにおいて深くネストしたジェネリクスは存在しない。

### スキーマビルダ

Dieselのエコシステムにおいて、[barrel](https://git.irde.st/spacekookie/barrel)のような素晴らしいライブラリがあり、SeaORMはスキーマを構築(`SeaQuery`)して管理([SeaSchema](https://github.com/SeaQL/sea-schema)するツールを持つ。
つまり、使い慣れたAPIと統一されたエクスペリエンスを意味する。

### 類似点

DieselとSeaORMの両者はスキーマファーストで、すべてがオブジェクト指向プログラムのクラスから始まるのではなく、既存のデータベーススキーマから始まることを意味している。

DieselとSeaORMの両者は関連があり、関連を定義することで複雑な結合をできることを意味している。

### 最後に

Rustエコシステムにおいて、Dieselはよく確立されたライブラリである。
SeaORMはとても新しい。
とちらも他を置き換えることはできない。
Rustコミュニティーが両方を強く成長させることを望む。
