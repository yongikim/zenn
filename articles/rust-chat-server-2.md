---
title: "Rust で同時接続可能なチャットサーバーを作る(2/n): 複数接続の実装"
emoji: "😀"
type: "tech"
topics: [Rust,TCP,tokio]
published: false
published_at: 2023-02-02
---
# 概要

Rustで複数接続可能なチャットサーバーを作る。
最終的には以下の機能を実装することを目指す。
- 複数クライアントの同時接続
- LINEのグループやSlackのチャンネルのようなチャットルーム機能
- WebSocket等による双方向通信

前回は、単一クライアントとの接続を実装した。
今回は、`tokio`を用いた並行化によって複数クライアントの同時接続を実装する。

# 前回までのコード

**Cargo.toml**
```toml:Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

**src/bin/chat-server.rs**
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

# `tokio::spawn()`を用いた並行化による同時接続の実現
複数クライアントの同時接続を可能にするため、`tokio::spawn()`を用いてタスクを生成する。  
`tokio:::spawn()`によって生成されたタスクは、`tokio`の非同期ランタイムによって並行に実行される。  
（昔書いた記事も読んでみてほしい）


`main`関数では次のように通信を処理していた。

**src/bin/chat-server.rs**
```rust:src/bin/chat-server.rs
let (socket, _addr) = listener.accept().await.expect("Failed to accept") {
accept_connection(socket).await;
```

この部分を次のように`loop`で囲い、`tokio::spawn()`によって通信の処理を並行化する。

**src/bin/chat-server.rs**
```rust:src/bin/chat-server.rs
loop {
    let (socket, _addr) = listener.accept().await.expect("Failed to accept");
    tokio::spawn(async { accept_connection(socket).await });
}
```

これによって、接続を受理すると即座に並行タスク`accept_connection(socket)`を生成した後、すぐさま次の`loop`に進んで次の接続を受理することができる。
もし、`tokio::spawn`を用いず、`loop`の中身を単純に`accept_connection(socket).await`とした場合、`loop`がブロックされ、後から接続したクライアントは先に接続したクライアントが切断されるまで待つはめになる。

このままでも良いが、あるクライアントとの接続に失敗したときプログラムが`panic`してしまうので、`while let`を用いて書き換えておく。

```rust:src/bin/chat-server.rs
while let Ok((socket, _addr)) = listener.accept().await {
    tokio::spawn(async { accept_connection(socket).await });
}
```

また、`accept_connection`関数の中身も、クライアントの接続状態を出力したり、無効なUTF-8シーケンス等に対応できるよう、少しだけ書き換えておく。

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
        match buf_reader.read_line(&mut line).await {
            Ok(bytes) => {
                if bytes == 0 {
                    println!("Close connection: {}", addr);
                    break;
                }
            }
            Err(error) => {
                println!("{error}");
                line = "Invalid UTF-8 detected\n".to_string();
            }
        }

        // ソケットへの書き込み（クライアントへの返信）
        writer.write_all(line.as_bytes()).await.unwrap();
        line.clear();
    }
}
```

ここまでで、サーバーは複数のクライアントと同時に会話できるようになったはずである。

# `nc` コマンドによる動作確認

コマンドウィンドウを2つ開き、それぞれのウィンドウで`nc`コマンドを実行してサーバーと接続する。  
サーバーに話しかけると、オウム返しをするだろう。

**ウィンドウ1**
```bash
nc localhost 8080
hello from client 1
hello from client 1
```

**ウィンドウ2**
```bash
nc localhost 8080
hello from client 2
hello from client 2
```

# 参考
- [Creating a Chat Server with async Rust and Tokio](https://www.youtube.com/watch?v=Iapc-qGTEBQ)

