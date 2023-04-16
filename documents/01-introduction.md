# イントロダクション

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

`SeaORM`において、テーブルのコレクションを持つデータベースは、`スキーマ`と呼ばれる。

`SeaORM`において、それぞれのテーブルは、関連するテーブルの`CRUD`操作を実行する、ある特定の[Entity](https://www.sea-ql.org/SeaORM/docs/generate-entity/entity-structure#entity)に対応する。

`Entity`トレイトは、実行時に[Column](https://www.sea-ql.org/SeaORM/docs/generate-entity/entity-structure#column)、[Relation](https://www.sea-ql.org/SeaORM/docs/generate-entity/entity-structure#relation)や[PrimaryKey](https://www.sea-ql.org/SeaORM/docs/generate-entity/entity-structure#primary-key)を検査するための`API`を提供する。

各テーブルは、`属性`として参照される複数の列を持つ。

これらの`属性`とそれらの値は、[Model](https://www.sea-ql.org/SeaORM/docs/generate-entity/expanded-entity-structure#model)と呼ばれる操作可能な`Rust`の構造体にグループ化される。

しかしながら、`Model`は読み込み操作のみ可能である。
挿入、更新及び削除するために、それぞれの属性のメタデータに付随する[ActiveModel](https://www.sea-ql.org/SeaORM/docs/generate-entity/expanded-entity-structure#active-model)を使用する必要がある。

最後に、`SeaORM`にはグローバルコンテキストとなるシングルトンは存在しない。
アプリケーションコードは、[DatabaseConnection](https://www.sea-ql.org/SeaORM/docs/install-and-config/connection)の所有権を管理する責任がある。
[Rocket](https://github.com/SeaQL/sea-orm/tree/master/examples/rocket_example)、[Actix](https://github.com/SeaQL/sea-orm/tree/master/examples/actix_example)及び[axum](https://github.com/SeaQL/sea-orm/tree/master/examples/axum_example)と統合する例を提供して、迅速に開始できるようにする。
