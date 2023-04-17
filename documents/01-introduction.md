# イントロダクション

- [イントロダクション](#イントロダクション)
  - [ORMとは](#ormとは)
  - [非同期プログラミング](#非同期プログラミング)
  - [SeaORMのコンセプト](#seaormのコンセプト)
  - [チュートリアルと例](#チュートリアルと例)

## ORMとは

Object Relational Mapper(ORM)は、オブジェクト指向プログラミング(OOP)言語とリレーショナルデータベースを相互作用させるプログラミングライブラリです。

データベース内のテーブルと列は、オブジェクトと属性にマッピングされ流一方で、追加されたメソッドはデータベースからデータを読み込み、またデータベースにデータを保存します。

Rustで構築されたサービスは軽量で（小さなバイナリサイズ、少ないメモリの使用）、安全で（コンパイル時の保証）、正確で（ユニットテストが十分に設計されている場合）、そして高速（コンパイル時の最適化がランタイムのオーバーヘッドを最小化する）です。

Rustは静的で、強く型付けされ、コンパイルされ、スレッドセーフで、ガベージコレクションを持たず、そして型破りなオブジェクト指向言語であるため、RustでORMを操作する方法は、既に親しんでいる他のスクリプト言語とは少し異なります。

SeaORMは、Rustでプログラミングする際の問題を回避しながら、上記の利点を享受することを試みています。

では、始めましょう。

## 非同期プログラミング

Rustにおける非同期プログラミングは最近開発され、Rust`1.39`で安定化されました。
非同期エコシステムは急速に進化しており、SeaORMは非同期のサポートを念頭に、1から構築された最初のクレートの1つです。

学べき最初の1つは[`Future`](https://www.sea-ql.org/SeaORM/docs/introduction/async#:~:text=learn%20is%20the-,Future,-trait.%20It%27s%20a)トレイトです。
Futureは、将来、計算して何らかの値を返却する関数のプレースホルダーです。
Futureは怠惰であるため、それは実際の作業が実施されるために`.await`が呼び出される必要があることを意味しています。
`Future`は、少しの努力で並行処理を実現させてくれます。
例えば、並行で複数のクエリを実行するための`future::join_all`があります。

2つ目に、Rustにおける`async`が、シンタックスシュガーを伴った[マルチスレッドプログラミング](https://rust-lang.github.io/async-book/03_async_await/01_chapter.html)であることです。
`Future`はスレッド間を移動するかもしれず、よってasync本体で使用される任意の変数は、[Send](https://doc.rust-lang.org/nomicon/send-and-sync.html)のようにスレッド間を移動できる必要があります。

3つ目に、`Rust`に複数の非同期ランタイムが存在することです。
間違いなく[actix](https://crates.io/crates/actix)、[async-std](https://crates.io/crates/async-std)、そして[tokio](https://crates.io/crates/tokio)は最も広く利用されている3つです。
SeaORMの基盤となるドライバである[SQLx](https://crates.io/crates/sqlx)は、それら3つすべてをサポートしています。

これらの概念を理解することは、非同期`Rust`を使いこなすために不可欠です。

## SeaORMのコンセプト

SeaORMにおいて、テーブルのコレクションを持つデータベースは、`Schema`と呼んでいます。

SeaORMにおいて、それぞれのテーブルは、関連するテーブルの`CRUD`（作成、読み込み、更新そして削除する）操作を実行する[Entity](https://www.sea-ql.org/SeaORM/docs/generate-entity/entity-structure#entity)に対応します。

`Entity`トレイトは、実行時にそのプロパティ[Column](https://www.sea-ql.org/SeaORM/docs/generate-entity/entity-structure#column)、[Relation](https://www.sea-ql.org/SeaORM/docs/generate-entity/entity-structure#relation)、そして[PrimaryKey](https://www.sea-ql.org/SeaORM/docs/generate-entity/entity-structure#primary-key)を検査する`API`を提供します。

それぞれテーブルは、`属性`として参照される複数の列を持ちます。

これらの`属性`とそれらの値は、それらを操作できる`Rust`の構造体（[Model](https://www.sea-ql.org/SeaORM/docs/generate-entity/expanded-entity-structure#model)）にグループ化されまます。

しかし、`Model`は読み込み操作のためでにあります。
挿入、更新または削除するために、それぞれの属性のメタデータを付与した[ActiveModel](https://www.sea-ql.org/SeaORM/docs/generate-entity/expanded-entity-structure#active-model)を使用する必要があります。

最後に、`SeaORM`にシングルトン（グローバルコンテキスト）はありません。
アプリケーションコードは、[DatabaseConnection](https://www.sea-ql.org/SeaORM/docs/install-and-config/connection)の所有権を管理する責任があります。
[Rocket](https://github.com/SeaQL/sea-orm/tree/master/examples/rocket_example)、[Actix](https://github.com/SeaQL/sea-orm/tree/master/examples/actix_example)、[axum](https://github.com/SeaQL/sea-orm/tree/master/examples/axum_example)、そして[poem](https://github.com/SeaQL/sea-orm/tree/master/examples/poem_example)と統合する例を提供して、早く開始できるようにします。

## チュートリアルと例

どのようにSeaORMを使用するかを説明した段階的なチュートリアルが必要であれば、オフィシャルな[SeaORMチュートリアル](https://www.sea-ql.org/sea-orm-tutorial/)を確認してください。
[コミュニテイ](https://github.com/SeaQL/sea-orm/blob/master/COMMUNITY.md#learning-resources)によって記述されたいくつかのチュートリアルもあります。

いくつかの頻繁にある質問と推奨された手法を[SeaORMクックブック](https://www.sea-ql.org/sea-orm-cookbook/)で確認することもできます。

もし、熱意があり、すぐに使えるものが必要な場合は、SeaQLがコミュニティが貢献したオフィシャルな例のセットを維持しています（歓迎します!）。

- [Actixの例](https://github.com/SeaQL/sea-orm/tree/master/examples/actix_example)
- [Axumの例](https://github.com/SeaQL/sea-orm/tree/master/examples/axum_example)
- [GraphQLの例](https://github.com/SeaQL/sea-orm/tree/master/examples/graphql_example)
- [jsonrpseeの例](https://github.com/SeaQL/sea-orm/tree/master/examples/jsonrpsee_example)
- [Poemの例](https://github.com/SeaQL/sea-orm/tree/master/examples/poem_example)
- [Rocketの例](https://github.com/SeaQL/sea-orm/tree/master/examples/rocket_example)
- [Salvoの例](https://github.com/SeaQL/sea-orm/tree/master/examples/salvo_example)
- [Tonicの例](https://github.com/SeaQL/sea-orm/tree/master/examples/tonic_example)

もし、広範囲にネストされた関連をクエリするWeb APIを構築する場合、GraphQLサーバーを構築することを検討してください。
[Seaography](https://www.sea-ql.org/Seaography/)は、SeaORMエンティティを使用してGraphQLリゾルバを構築するGraphQLフレームワークです。
詳しくは、"[Seaography入門](https://www.sea-ql.org/blog/2022-09-27-getting-started-with-seaography/)"を読んでください。
