---
title: "Rustのstd::mem::replaceの使い方"
emoji: "🦀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Rust]
published: true
---

## 背景
[このドキュメント](https://rust-unofficial.github.io/too-many-lists/second-option.html)に沿ってRustで単連結リストを書いていて、リストに要素を追加する際、既存のリストの先頭要素を新しく追加する要素の`next`要素に付け替えようとしたところ、以下のようなエラーが発生した。

> cannot move out of `self.head` which is behind a mutable reference

```rust
use std::mem;

pub struct List {
    head: Link,
}

type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        Self { head: None }
    }

    // `List` の先頭に `Node` を追加する。
    pub fn push(&mut self, elem: i32) {
        let node = Box::new(Node {
            elem,
            // cannot move out of `self.head` which is behind a mutable reference
            next: self.head,
        });

        self.head = Some(node);
    }
}
```

エラーメッセージによると、可変参照（`&mut`）を通じての所有権移動が許可されていないということらしい。
要するに、参照である`self`からその要素をmoveすることはできないということである。
`self.head.clone()`としても良いが、このようなケースにおいて一般に対象が`clone`を実装しているとは限らないので、いつでも`clone`できるとは限らない。

## 本題
これはよく知られたエラーらしく、`std::mem::replace`を使うことで解決できる。

```rust
pub fn push(&mut self, elem: i32) {
    let node = Box::new(Node {
        elem,
        next: mem::replace(&mut self.head, None),
    });

    self.head = Some(node);
}
```

`replace`は以下のようなシグネチャを持っており、`dest`の値を`src`で書き換え、書き換える前の`dest`の値を返す関数である。

```rust
pub fn replace<T>(dest: &mut T, src: T) -> T
```


[replaceの実装](https://doc.rust-lang.org/src/core/mem/mod.rs.html#911)を見てみよう。非常にシンプルで、説明の余地はほとんどなさそうである。

```rust
pub const fn replace<T>(dest: &mut T, src: T) -> T {
    // SAFETY: We read from `dest` but directly write `src` into it afterwards,
    // such that the old value is not duplicated. Nothing is dropped and
    // nothing here can panic.
    unsafe {
        let result = ptr::read(dest);
        ptr::write(dest, src);
        result
    }
}
```

唯一、戻り値の所有権について気になるところだが、 [ptr::readのドキュメント](https://doc.rust-lang.org/std/ptr/fn.read.html#ownership-of-the-returned-value)によると、`ptr::read<T>`は

> `read` creates a bitwise copy of `T`

と書いてあるので、readの戻り値は所有権付きであることがわかる。

ちなみに、`read(&s)`は`s`と同一のメモリを指すため、`read`の戻り値と`s`を同時に使用するのは危険である旨が書かれている。
具体的には、`let s2 = read(&s)`とした後に`s2`に他の値を再代入すると、根底にあるメモリは`drop`されてしまうため、それ以降`s`に再代入しようとすると根底にあるメモリに対して再び`drop`が呼ばれることになり、未定義動作を引き起こす。

ちなみにちなみに、今回の例のように`Option`を使う場合には、[`take`](https://doc.rust-lang.org/src/core/option.rs.html#1696)というラッパー関数が使える。

```rust
pub const fn take(&mut self) -> Option<T> {
      // FIXME replace `mem::replace` by `mem::take` when the latter is const ready
      mem::replace(self, None)
}
```

```rust
pub fn push(&mut self, elem: i32) {
    let node = Box::new(Node {
        elem,
        next: self.head.take(),
    });

    self.head = Some(node);
}
```
