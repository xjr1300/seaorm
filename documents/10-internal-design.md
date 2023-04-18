# 内部実装

- [内部実装](#内部実装)
  - [トレイトと型](#トレイトと型)
    - [エンティティ](#エンティティ)
    - [列](#列)
    - [プライマリーキー](#プライマリーキー)
    - [モデル](#モデル)
    - [アクティブモデル](#アクティブモデル)
    - [アクティブ列挙型](#アクティブ列挙型)
    - [関連（Relation）](#関連relation)
    - [関連した（Related）](#関連したrelated)
    - [リンクした（Linked）](#リンクしたlinked)
  - [導出マクロ](#導出マクロ)
    - [エンティティモデル](#エンティティモデル)
    - [エンティティ](#エンティティ-1)
    - [列](#列-1)
    - [プライマリーキー](#プライマリーキー-1)
    - [モデル](#モデル-1)
    - [アクティブモデル](#アクティブモデル-1)
    - [関連](#関連)
    - [イテラブル](#イテラブル)
  - [Dieselとの比較](#dieselとの比較)
    - [アーキテクチャ](#アーキテクチャ)
    - [プログラミングパラダイム](#プログラミングパラダイム)
    - [スキーマビルダ](#スキーマビルダ)
    - [類似点](#類似点)
    - [最後に](#最後に)

## トレイトと型

### エンティティ

[EntityTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)トレイトを実装したユニット構造体は、データベースのテーブルを表現します。

このトレイトは、次のエンティティの属性を含んでいます。

- テーブル名（[EntityName](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)を実装）
- 列（[ColumnTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)を実装）
- 関連（[RelationTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)を実装）
- プライマリーキー（[PrimaryKeyTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)と[PrimaryKeyToColumn](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)を実装）

このトレイトは、CRUD操作するAPIも提供しています。

- 選択: `find`、`find_*`
- 挿入: `insert`、`insert_*`
- 更新: `update`、`update_*`
- 削除: `delete`、`delete_*`

### 列

[ColumnTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)トレイトを実装した列挙型は、テーブルのすべての列と列の型と属性を表現します。

このトレイトは、次を実装します。

- [IdenStatic](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)は、静的ライフタイムで列の識別子とのマッピングを提供しています。
- [Iterable](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)は、SeaORMコアにすべての列のバリアントを反復させます。

### プライマリーキー

[PrimaryKeyTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)トレイトを実装した列挙型は、プライマリーキーを表現します。
それぞれのプライマリーキーのバリアントは、列のバリアントとの対応する必要があります。

このトレイトは、次を実装します。

- [IdenStatic](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)は静的ライフタイムでプライマリーキーの識別子とのマッピングを提供します。
- [Iterable](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)はSeaORMコアにすべてのプライマリーキーのバリアントを反復させます。

### モデル

[ModelTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)を実装した構造体は、クエリ結果をメモリに保存します。
これは、読み込み目的で使用されることを意図しており、ステートレスです。

このトレイトは、次を実装します。

- [FromQueryResult](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)は、プリミティブな問い合わせの結果を対応するモデルに変換します。

### アクティブモデル

[ActiveModelTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)を実装した構造体は、挿入／更新操作を表現します。
これは、編集され、データベースに保存されることを意図しています。

このトレイトは、次を実装します。

- [ActiveModelBehavior](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)は、アクティブモデルの様々な操作のハンドラを定義します。

### アクティブ列挙型

[ActiveEnum](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)を実装した列挙型は、Rustの列挙型のバリアントとしてデータベースに保存される値を表現します。

### 関連（Relation）

[RelationTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)を実装した列挙型は、他のエンティティとの関連を定義します。

このトレイトは、次を実装します。

- [Iterable](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)は、SeaORMコアにすべての関連のバリアントを反復させます。

### 関連した（Related）

[Related](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)はジェネリックトレイトで、互いに関連付けられたエンティティ同士を問い合わせする結合パスを定義して、特に多対多関連で便利です。

### リンクした（Linked）

[Linked](https://www.sea-ql.org/SeaORM/docs/internal-design/trait-and-type#)トレイトは、連鎖した関連、自己参照関連、そして2つのエンティティ間の複数の関連を含んだ、複雑な結合パスを定義します。

## 導出マクロ

### エンティティモデル

[EntityModel](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)導出マクロは、与えられた`Model`から`Entity`、`Column`および`PrimaryKey`を自動的に生成する、'全能な'マクロです。

### エンティティ

`DeriveEntity`導出マクロは、`Entity`に[EntityTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro/#)を実装して、現在のスコープ内に存在する`Model`、`Column`、`PrimaryKey`及び`Relation`を仮定します。
また、それは、`Entity`のために[Iden](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)と[IdenStatic](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)の実装を提供します。

### 列

[DeriveColumn](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)派生マクロは、`Column`に[ColumnTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)を実装します。
それは、[Iden](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)と[IdenStatic](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)を実装することにより、それぞれの列の識別子を定義します。
また、[EnumIter](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)も導出されて、すべての列挙型のバリアントを反復させます。

### プライマリーキー

[DerivePrimaryKey](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)導出マクロは、`PrimaryKey`にプライマリーキーと列間の面倒なマッピングを定義する[PriamryKeyToColumn](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)を実装します。
また、[EnumIter](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)も導出されて、すべての列挙型のバリアントを反復させます。

### モデル

[DeriveModel](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)導出マクロは、`Model`にモデルのすべての属性のセッターとゲッターを提供する`ModelTrait`を実装します。
また、問い合わせ結果を対応する`Model`に変換する[FromQueryResult](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)も実装します。

### アクティブモデル

[DeriveActiveModel](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)導出マクロは、`ActriveModel`にアクティブモデル内のすべてのアクティブ値のセッターとゲッターを提供する[ActiveModelTrait]を実装します。

### 関連

[DeriveRelation](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)導出マクロは、`Relation`に[RelationTrait](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)を実装します。

### イテラブル

[EnumIter](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)導出マクロは、列挙型のすべてのバリアントを反復するために[Iterable](https://www.sea-ql.org/SeaORM/docs/internal-design/derive-macro#)を実装します。

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
