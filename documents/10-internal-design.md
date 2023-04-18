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
  - [アーキテクチャ](#アーキテクチャ-1)

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

[SeaORM](https://github.com/SeaQL/sea-orm)と[Diesel](https://github.com/diesel-rs/diesel)は、データベースの完全なインターフェースを提供するという、同じゴールを共有しています。

SeaORMとDieselの両方は、MySQL、PostgreSQLおよびSQLiteで動作するため、どちらかを選択する必要はありますせん。
Dieselは、サードパーティがカスタムなバックエンドを実装させる一方で、SeaORMはできません。

SeaORMがDieselとは異なる選択をしたモノがあります。

### アーキテクチャ

最初に、おそらく最も要求された機能は、非同期Rustのサポートです。
現在、非同期の使用が良いパフォーマンスを提供しないかもしれない一方で、非同期プログラミングは早期にする必要があるアーキテクチャの決定です。
SeaORMを選択することにより、Rustの非同期エコシステムが成熟することが待ち遠しくなります。

内部で[SQLx](https://crates.io/crates/sqlx)を持つSeaORMは、純粋なRustの技術スタックを提供します。
Dieselはデフォルトでネイティブドライバを使用しているため、それを純粋なRustのドライバに置き換えることは、いくらかの努力が必要です。
どちらにも利点と欠点があるため、どちらを使用するかは好み次第です。

SeaORMはモジュラーデザインを採用しています。
もし、ORMのアイデアが好きでない場合、クエリビルダーを基盤とした[SeaQuery](https://crates.io/crates/sea-query)を引き続き利用することは間違いありません。
SeaORMは軽量で容易に任意のプロジェクトに統合できます。
SeaORMを使用する場合、SeaORMのAPIも利用できるため、必要であれば柔軟なクエリビルダの能力で、高レベルの抽象化の利点を得られます。

### プログラミングパラダイム

同期と非同期の基盤に加えて、DieselとSeaORMには静的と動的といった大きな違いがあります。

Dieselは、型が完全に静的にチェックされる完全コンパイルタイムAPIを提供します。
Dieselでも動的な問い合わせを実行できますが、コンパイル時型検査のいくつかの欠点を失うことがあります。

SeaORMは動的で、モノがランタイムで確立されます。
SeaORMはより柔軟です。
いくつかのコンパイル時の検査を失っても、SeaORMは代わりにテストで正確さを証明することを支援します。

両方のライブラリはトレイトやジェネリクスを多用していますが、SeaORMはより少ない型を生成します。
Dieselのそれぞれの列は（さまざまなトレイトを実装した）構造体で、SeaORMのそれぞれの列は列挙体（エンティティのすべての列をまとめた1つの列挙体を形成して、それはいくつかのトレイトを実装した）のバリアントです。

SeaORMにおいて、深くネストしたジェネリクス（例えば、`A<B<C<D<E>>>>`）は存在しません。

### スキーマビルダ

Dieselのエコシステムに[barrel](https://git.irde.st/spacekookie/barrel)のような素晴らしいライブラリがある一方で、SeaORMはスキーマを構築（SeaQuery）して管理（[SeaSchema](https://github.com/SeaQL/sea-schema)）するツールが維持されています。
それは、親和性のあるAPIと統一された体験を意味します。

### 類似点

DieselとSeaORMの両者はスキーマファーストであることで、OOPなクラスから開始する代わりに、すべてが既存のデータベーススキーマから開始することを意味します。

DieselとSeaORMの両方はリレーショナルであり、関連を定義することで複雑な結合をできることを意味しています。

### 最後に

Dieselは、Rustエコシステムにおいて、十分に確立されたライブラリです。
SeaORMはとても新しい。
とちらも他を置き換えることはできません。
Rustコミュニティーが両方を強く成長させることを望んでいます。

## アーキテクチャ

> 海の下に飛び込みましょう🤿

![SeaORMのアーキテクチャ](https://www.sea-ql.org/SeaORM/img/SeaORM%20Architecture.svg)

SeaORMのアーキテクチャを理解するために、ORMとは何か議論しましょう。
ORMは、データベースに対して実施する一般的な操作の抽象化を提供して、実際のSQL問い合わせの詳細な実装を隠すために存在します。

良いORMでは、APIの表面の下を見て悩む必要はありません。
それをするまでは。
*'抽象化が漏れる'*ということを聞いたことがありますが、まさにそれがそうです。

SeaORMが採用した方法は、**'階層化された抽象化（layered abstraction）'**で、必要に応じて1つ下のレイヤを掘り下げます。
これが、SeaORMを独立したリポジトリにした理由です。
それは、公開APIの表面と分離した名前空間を使用するとそれ自身で役に立ち、混乱する内部APIを作成することが、モノリシックアプローチよりもはるかに困難になります。

SeaORMの中心的なアイデアは、すべてがランタイム設定に近いです。
コンパイル時、SeaORMはどのデータベースに基づいているかを知りません。

データベース不可知論者がもたらす利点はなんでしょうか?
例えば、次をできます。

1. ランタイム設定に依存して、どのデータベースでもアプおりを機能させることができる
2. 同じモデルを使用して、それらを異なるデータベース間に転送する
3. 'データ構造体クレート'を作成することにより、異なるプロジェクト間でエンティティを共有して、下流の'振る舞いクレート'によってデータベースを選択できる

SeaORMのAPIは薄いシェルではありませんが、レイヤで構成されており、その下のそれぞれの例あやはそれほど抽象的ではありません。

APIが利用されるときに、ことなるステージがあります。

よって、SeaORMコードベースを案内する**ステージ**と**抽象**という2つの次元があります。

最初は宣言ステージです。
エンティティとそれらにまたがる関連は、`EntityTrait`、`ColumnTrait`と`RelationTrait`などで定義されています。

2番目は問い合わせ構築ステージです。

最上位レイヤは、`Entity`、`find*`、`insert`、`update`と`delete`メソッドで、直感的に基本的なCRUD操作を実行できます。

1つ下のレイヤは、`Select`、`Insert`、`Update`と`Delete`構造体で、それらはそれぞれ自身のより高度なAPIを持っています。

1つ下のレイヤは、SeaQueryの`SelectStatement`、`InsertStatement`、`UpdateStatement`と`DeleteStatement`で、それらはSQL構文木を扱う豊富なAPIを持っています。

3番目は実行ステージです。
`Selector`、`Inserter`、`Updater`と`Deleter`の分離された構造体の集合は、データベースコネクションに対して文を実行する責任があります。

最後は、Rust型に問い合わせ結果が変換され、構造体に入れられたときの解決ステージです。
その後、もし、それが関係問い合わせの場合、構造体はそれらの関連に従ってつなぎあわされます。

実行と解決ステージのみがデータベース特有であるため、それらを置き換えるために異なるドライバを使用する可能性を持ちます。

いつの日か、さまざまな構文、プロトコルそしてフォームファクターの行列を備えた多くのデータベースをサポートするようになると考えています。
