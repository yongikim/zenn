---
title: "Rust で複数の非同期処理を並行的に実行する"
emoji: "😀"
type: "tech"
topics: [Rust,非同期処理]
published: true
published_at: 2021-10-24
---
Rust では、 `fn` の代わりに `async fn` を用いることで、非同期関数を定義することができます。非同期関数は、呼び出し元のスレッドをブロックせず、並行的に実行されます。

```rust
// 通常の関数
fn hello_world_sync() {
	println!("Hello world");
}

// 非同期関数
async fn hello_world_async() {
	println!("Hello world");
}
```

Rust の非同期関数を、通常の関数と同じように `()` をつけて呼び出すと、戻り値として `impl Future` が得られますが、これに対して待機 (`await`) しない限り、処理は行われません。また、非同期処理を扱うためには非同期ランタイムが必要となりますが、実装が重いなどの理由から、Rust の標準ライブラリには含まれていません。そのため、tokio や async-std などの外部ライブラリを用いる必要があります。以下では、デファクトスタンダードである tokio を用いて説明します。

tokio を用いて `hello_world_async` を実行するコードは、次のようになります。

```Rust
#[tokio::main]
async fn main() {
	let future = hello_world_async(); // 何も出力されない
    future.await; 					  // "Hello world"
}
```

`#[tokio::main]`マクロによって、非同期ランタイムの初期化が行われ、`main` 関数の中身が実行されます。4行目で `future` を `await` することで、`future` の完了を待機します。

複数の非同期処理を並行的に実行するためには、次のように複数の `future` を定義し、futures クレートの`join_all` を使って全ての処理の完了を待機します。

```rust
use futures::future::*;
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
	let handles = (0..10).map(|n| {
        tokio::spawn(async move {
            let dur_ms = (10 - n) as u64 * 1000;
            sleep(Duration::from_millis(dur_ms)).await;
            println!("{}", n);
        })
    });
    
    join_all(handles).await;
}
```

`tokio::spawn` は、`async` ブロックからタスクを生成します。上記のコードを実行すると、10個のタスクが生成され、tokio ランタイムによってバックグラウンドで並行的に実行されます。その結果、9, 8, 7, ..., 0 の順に1秒おきに出力され、最終的に次のような出力が得られるはずです。

```shell
9
8
7
6
5
4
3
2
1
0
```

以上で、複数の非同期処理を並行的に実行することができました。チャネルなど他にも重要な概念はありますが、これを応用して、リクエスト毎に個別の task を生成するようにすることで、高速な web サーバ等を作ることができます。

