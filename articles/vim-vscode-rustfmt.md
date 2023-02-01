---
title: "vimやvscodeでrustfmtによる自動整形が効かないとき"
emoji: "😀"
type: "tech"
topics: [Vim,Rust,VSCode,rustfmt]
published: false
---
## 症状
vimやvscodeでファイル保存時に`rustfmt`でコードが自動整形されるよう設定しているにも関わらず、コードが整形されなかった。
コマンドは発行されているので、エディタの問題ではなさそうだった

## 解決策
ログを見ると、以下のように出力されており、`rustfmt`に`--edition 2018`以上を指定する必要があるようだった。
```
  |
6 | async fn main() -> Result<(), Error> {
  | ^^^^^ to use `async fn`, switch to Rust 2018 or later
  |
  = help: pass `--edition 2021` to `rustc`
  = note: for more on editions, read https://doc.rust-lang.org/edition-guide
``` 
[公式ドキュメント](https://rust-lang.github.io/rustfmt/?version=v1.5.1&search=)を見ると、プロジェクトの親ディレクトリや`$HOME`に`rustfmt.toml`等の名前のファイルを作成し、エディションを指定すれば良いらしい。  
というわけで以下のファイルを作成すると無事`rustfmt`が動くようになった。
```toml:$HOME/rustfmt.toml
edition = 2018
```

