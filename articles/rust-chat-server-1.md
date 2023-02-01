---
title: "Rust で同時接続可能なチャットサーバーを作る (1/n): 単一クライアントとの接続"
emoji: "😀"
type: "tech"
topics: [Rust,Web,websocket,chat,tokio]
published: false
published_at: 2023-01-31
---
# 概要

Rustで複数接続可能なチャットサーバーを作る。
最終的には以下の機能を実装することを目指す。
- 複数クライアントの同時接続（[次回記事](https://zenn.dev/yongikim/articles/rust-chat-server-2)）
- LINEのグループやSlackのチャンネルのようなチャットルーム機能
- WebSocket等による双方向通信

この記事では、まずはローカルなTCPソケット通信を用いて単一クライアントとの接続を実装する。

# ソケット通信の待機

`tokio` を用いて非同期処理を実装するので、`cargo init` でプロジェクトを作成した後、`Cargo.toml` に以下を追記する。

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

ソケット通信を待機するコードは次のようになる。
`127.0.0.1:8080` で通信を待機し、実際の通信の処理は `accept_connection` 関数に任せている。

```rust:src/bin/chat-server.rs
use std::{env, io::Error};
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::{TcpListener, TcpStream};

#[tokio::main]
async fn main() -> Result<(), Error> {
    // コマンドライン引数からアドレスを取得する
    // デフォルトは 127.0.0.1:8080
    let addr = env::args()
        .nth(1)
        .unwrap_or_else(|| "127.0.0.1:8080".to_string());

    // リスナーを作成する
    let socket = TcpListener::bind(&addr).await;
    let listener = socket.expect("Failed to bind");
    println!("Listening on: {}", addr);

    // 通信を処理する
    let (mut stream, _addr) = listener.accept().await.expect("Failed to accept");
    accept_connection(stream).await;

    Ok(())
}

async fn accept_connection(mut stream: TcpStream) {
    unimplemented!();
}
```

次のコマンドでサーバーを起動する。

```bash
cargo run --bin chat-server

# Listening on: 127.0.0.1:8080
```

サーバーと通信するには、`telnet` コマンドを用いる。

```bash
telnet 127.0.0.1 8080
 
# Trying 127.0.0.1...
# Connected to localhost.
# Escape character is '^]'.
```

`accept_connection`が未実装なので、接続が確立した瞬間にサーバーが `panic` してしまう。
通信を行うためには `accept_connection` 関数を実装する必要がある。

```rust:src/bin/chat-server.rs
async fn accept_connection(mut stream: TcpStream) {
    // クライアントのアドレスの表示
    let addr = stream
        .peer_addr()
        .expect("connected streams should have a peer address");
    println!("Peer address: {}", addr);

    // ソケットを読み込み部と書き込み部に分割
    let (reader, mut writer) = stream.split();

    // 文字列への読み込み
    let mut buf_reader = BufReader::new(reader);
    let mut line = String::new();
    buf_reader.read_line(&mut line).await.unwrap();

    // ソケットへの書き込み（クライアントへの返信）
    writer.write_all(line.as_bytes()).await.unwrap();
}
```

ここまでで、単一のクライアントと一回だけやりとりを行うことができるようになった。
サーバーを起動する。

```bash
cargo run --bin chat-server

# Listening on: 127.0.0.1:8080
```

`telnet` コマンドでサーバーに接続する。

```bash
telnet 127.0.0.1 8080

# Trying 127.0.0.1...
# Connected to localhost.
# Escape character is '^]'.
```

コマンドを打ったウィンドウでそのまま文字列を入力してEnterを押すと、サーバーへ送信することができる。
サーバーから返事が返ってきたら成功である。
サーバー側ではクライアントのアドレスが表示されている。

最後に文字列への読み込みの部分を`loop`で囲えば、サーバーと繰り返し通信を行うことができる。
なお、クライアントへの返信を書き込んだ後毎回 `line.clear()` を呼んで文字列の内容をクリアする必要がある。

```rust:src/bin/chat-server.rs
async fn accept_connection(mut stream: TcpStream) {
    // クライアントのアドレスの表示
    let addr = stream
        .peer_addr()
        .expect("connected streams should have a peer address");
    println!("Peer address: {}", addr);

    // ソケットを読み込み部と書き込み部に分割
    let (reader, mut writer) = stream.split();

    // 文字列への読み込み
    let mut buf_reader = BufReader::new(reader);
    let mut line = String::new();
    loop {
        buf_reader.read_line(&mut line).await.unwrap();

        // ソケットへの書き込み（クライアントへの返信）
        writer.write_all(line.as_bytes()).await.unwrap();
        line.clear();
    }
}
```

[次回](https://zenn.dev/yongikim/articles/rust-chat-server-2)は複数クライアントの同時接続に対応する。

# 参考
- [Creating a Chat Server with async Rust and Tokio](https://www.youtube.com/watch?v=Iapc-qGTEBQ)

