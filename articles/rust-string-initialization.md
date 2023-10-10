---
title: "Rust は文字列をどう生成するか"
emoji: "😀"
type: "tech"
topics: [Rust,文字列]
published: true
published_at: 2023-10-11
---
## 文字列の生成
Rustには`String`という、文字列を扱う型がある。
その実態は`vec`という`Vec<u8>`型の唯一のフィールドを持つ`struct`である。

```rust
struct Rust {
  vec: Vec<u8>
}
```

これは、Rustのデフォルトの文字コードであるUTF-8において、可変長のバイト列によってUnicodeのコードポイントを表現することに由来している。

`String`を初期化するとき、このようなコードをよく書く。

```rust
let str = String::from("hello, world!");
```

`String::from`は実際のところ何をしているのだろうか？

```rust
impl From<&str> for String {
    /// Converts a `&str` into a [`String`].
    ///
    /// The result is allocated on the heap.
    #[inline]
    fn from(s: &str) -> String {
        s.to_owned()
    }
}
```

つまり、`str`に実装されている`to_owned()`を見れば良い。

```rust
impl ToOwned for str {
    type Owned = String;
    fn to_owned(&self) -> String {
      unsafe { String::from_utf8_unchecked(self.as_bytes().to_owned()) }
    }
}
```

ここでは、文字列スライス`str`をバイト列に変換してから`String`に変換している。
`String::from_utf8_unchecked`はその名の通り、渡されたバイト列がUTF-8として有効かどうかをチェックしないが、ここでは`str`から変換しているので有効性は保証されており、無駄な計算を避けるためにもチェックをスキップする方が適切である。

```rust
pub const unsafe fn from_utf8_unchecked(v: &[u8]) -> &str {
    // SAFETY: the caller must guarantee that the bytes `v` are valid UTF-8.
    // Also relies on `&str` and `&[u8]` having the same layout.
    unsafe { mem::transmute(v) }
}
```

チェックを行う必要がある場合は`String::from_utf8`を使う。
`str::from_utf8`を使ってUTF-8としての有効性を確認している。
バイト列がUTF_8として無効な場合は`Error`を返すようになっている。

```rust
pub fn from_utf8(vec: Vec<u8>) -> Result<String, FromUtf8Error> {
  match str::from_utf8(&vec) {
    Ok(..) => Ok(String { vec }),
    Err(e) => Err(FromUtf8Error { bytes: vec, error: e }),
  }
}

```

ちなみに、`from_utf8(bytes)`が無効なバイト列を受け入れないのに対して、`from_utf8_lossy(bytes)`はバイト列の無効な部分を`U+FFFD`(REPLACEMENT CHARACTER)に変換する。
後者の実現においては、`Cow`という興味深い型が活躍している。（別記事作成予定）
簡単に言うと、データの参照と所有のenumであり、所有が必要になったときに初めてアロケーションを行って所有を得るという型である。

## 文字列の連結
文字列の連結をどのように行うか。
例えばrubyだと下記のように愚直に書くことが多いと思うが、この方法では文字列を追加するたびにアロケートが発生し、効率が悪い。
（簡単のため結合する文字列の長さがすべてxであるとし、k回結合するとすると、計算量はO(xk^2)となる。)

```ruby
str = "hello0"
str_added = 1..1000.map { |i| " hello#{i} " }
str_added.each { |s| str += s }
puts str
# "hello0 hello1 hello2 ... hello999"
```

そこで、多くの言語において、文字列クラスは内部にバッファを持っており、文字列を連結する際には専用の関数を呼び出してバッファに文字列を追加していく形式をとっている。
Rustも例外ではなく、`String`の`vec`フィールドがバッファにあたり、`push_str`関数を用いてバッファに文字列を追加することで文字列を効率よく連結することができる。
バッファの容量が十分大きければ、計算量はO(xk)で済む。

```rust
pub fn push_str(&mut self, string: &str) {
    self.vec.extend_from_slice(string.as_bytes())
}
```

ここで`extend_from_slice` の内部では、`vec`のキャパシティをチェックし、結合後の文字列の長さがキャパシティを超える場合はその分だけ追加でアロケーションを行った上で、末尾に文字列を追加するという処理が行われる。

```rust
// In Vec's implementation
fn extend_desugared<I: Iterator<Item = T>>(&mut self, mut iterator: I) {
    while let Some(element) = iterator.next() {
        let len = self.len();
        if len == self.capacity() {
            let (lower, _) = iterator.size_hint();
            self.reserve(lower.saturating_add(1));
        }
        unsafe {
            ::std::ptr::write(self.get_unchecked_mut(len), element);
            // NB can't overflow since we would have had to alloc the
            // address space
            self.move_tail(1);
        }
    }
}
```

つまり、最初に確保したキャパシティが足りなくなった後は、文字列を結合するたびに、追加部分の長さ分だけアロケーションが行われることになる。
それでも都度全体をアロケーションするような最初の例よりはかなり効率は良くなるだろう。

#### 以下補足
`str`の`as_bytes`からの`to_onwed()`で`Vec<u8>`がアロケートされる。

```rust
impl str {
  pub const fn as_bytes(&self) -> &[u8] {
      // SAFETY: const sound because we transmute two types with the same layout
      unsafe { mem::transmute(self) }
  }  
}

// slice ここで [T] は、Clone を実装する型 T のスライス
// to_vec() によって Vec がアロケートされる
impl<T: Clone> ToOwned for [T] {
  fn to_owned(&self) -> Vec<T> {
      self.to_vec()
  }  
}
```

`with_capacity`も見ておく必要がある。じゃあ上のfrom_utf8だとcapacityの扱いはどうなっているのか？

```rust
fn pub with_capacity
```

`str::from_utf8`の実装の一部を見てみる

```rust
fn pub from_utf8
```

