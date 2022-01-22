# イントロダクション

## ORMとは

`Object Relational Mapper(ORM)`は、オブジェクト指向プログラミング(OOP)言語とリレーショナルデータベースを相互作用させるプログラミングライブラリである。

データベース内のテーブルと列は、オブジェクトと属性にマッピングされ、追加されたメソッドはデータベースからデータを読み込み、またデータベースに保存する。

`Rust`に組み込まれたサービスは、小さなバイナリサイズで少量のメモリを使用するなど軽量で、コンパイル時に型保証されるなど安全で、ユニットテストが適切であれば正確で、コンパイル時の最適化はランタイムのオーバーヘッドを最小化するなど高速である。

`Rust`が静的で、強く型付けされ、コンパイルされ、スレッドセーフで、ガベージコレクションを持たず、型破りなオブジェクト指向言語であるため、`Rust`でORMの作業をすることは、慣れたスクリプト言語とは少し異なる。

`SeaORM`は、`Rust`でプログラミングする際の問題を回避しながら、上記の利点を享受できる様にする。

## 非同期プログラミング

`Rust`における非同期プログラミングは最近開発され、`Rust 1.39`で安定した。
非同期エコシステムは急速に進化しており、`SeaORM`は非同期のサポートを念頭にした、0から構築された最初のクレートの1つである。

学べき最初の1つは[Future](https://www.sea-ql.org/SeaORM/docs/introduction/async#:~:text=learn%20is%20the-,Future,-trait.%20It%27s%20a)トレイトである。
`Future`は、将来に計算して何らかの値を返却する関数のプレースホルダーである。
`Future`は遅延評価されるため、実際の作業をするためには、`await`を呼び出す必要がある。
`Future`はプログラミング労力をほとんどかけずに並行処理を実現できる。
例えば、`future::join_all`で並行に複数のクエリを実行できる。

2つ目は`async`で、`Rust`の`async`はシンタックスシュガーを使用した[マルチスレッドプログラミング](https://rust-lang.github.io/async-book/03_async_await/01_chapter.html)である。
`Future`はスレッド間を移動して、非同期の本体内で使用されるいくつかの変数は、[Send](https://doc.rust-lang.org/nomicon/send-and-sync.html)のようにスレッド間を移動できるうようにする必要がある。

3つ目は、`Rust`に複数の非同期ランタイムが存在することである。
間違いなく[actix](https://crates.io/crates/actix)、[async-std](https://crates.io/crates/async-std)と[tokio](https://crates.io/crates/tokio)は最も広く利用されている3つである。
`SeaORM`の基盤となるドライバである[SQLx](https://crates.io/crates/sqlx)、それら3つすべてをサポートしている。

これらの概念を理解することは、非同期に`Rust`を実行するために不可欠である。

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
