---
title: "あの日見た匿名ライフタイム"
emoji: "💐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
---

プライベートで Rust を書くようになってから 4 ヶ月ほど経ちます。

多くの初心者と同じようにライフタイムで苦しめられています。最近も**匿名ライフタイム**のことがよく分からず、いくつかプログラムを書いたりドキュメントや Web で情報を漁って調べていました。調べはじめた当初よりは理解も進んで、気をつけるポイントも分かってきたので記事にまとめておきます。

## 匿名ライフタイムとは何か？

まずは「匿名ライフタイムとは何か？」について説明します。

例として、次の構造体を見てください。

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}
```

これは [The Rust Programming Language](https://doc.rust-lang.org/book/title-page.html) にも出てくる例 [^1] なので見たことがある人も多いでしょう。*ImportantExcerpt* 構造体は文字列への参照を持ち、`'a` という名前のついたライフタイムを持っています。この構造体は以下のようにして使うことができます。

```rust
fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().expect("Could not find a '.'");
    let i = ImportantExcerpt {
        part: first_sentence,
    };

    println!("part = {}", i.part); // => part = Call me Ishmael
}
```

もちろん、`'a` というライフタイムは名前がついているので匿名ライフタイムではありません。

では、このコードを少し改変します。「最初のセンテンスを切り出して *ImportantExcerpt* 構造体を返す」処理を関数に切り出してみましょう。

```rust
fn first_sentence(novel: &str) -> ImportantExcerpt<'_> {
    let first_sentence = novel.split('.').next().expect("Could not find a '.'");
    ImportantExcerpt {
        part: first_sentence,
    }
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let i = first_sentence(&novel);

    println!("part = {}", i.part); // => part = Call me Ishmael
}
```

`first_sentence` 関数のシグネチャに注目です。

```rust
fn first_sentence(novel: &str) -> ImportantExcerpt<'_>
```

返り値の *ImportantExcerpt* 構造体には `'_` というライフタイムが指定されており、識別子による名前がついていません。このように名前がついていないライフタイムを**匿名ライフタイム**といいます。





---

**記事にするときの構成**

- はじめに
  - プライベートで Rust を書き始めて数ヶ月
    - ライフタイムに苦しめられている
  - 匿名ライフタイムについてよく分からなかったので調べた
    - そもそも何なのか？
    - 何のためにあるのか？
    - 日々のプログラミングでどのように役立つのか？
- 匿名ライフタイムとは何か？
  - 匿名ライフタイムの例
    - 適当なプログラム
    - 標準ライブラリの例
    - `impl Trait` を返すときの例
  - Edition 2018 で導入された詳細
    - これらをちゃんと読めば詳細は理解できる
    - 何のために導入されて、どう役立つのかを中心に書きます
- 匿名ライフタイムの意義と効用
  - 関数での匿名ライフタイム
    - ライフタイムの省略ルールをおさらい
    - 省略されていることを明示するための匿名ライフタイム
    - 何が嬉しいのか？
      - 参照を持つ構造体を返す例
      - Edition 2018 での新しい警告
  - `impl` ブロックでの匿名ライフタイム
    - Edition 2018 で導入された
    - Trait を実装するときに便利
  - Trait object の匿名ライフタイム
    - `impl Trait` を返すときの例再掲
    - ライフタイムの省略ルール再び
    - 匿名ライフタイムでルールを変える
- まとめ
  - 構造体が他の参照を借用していることを示すのはいいこと
    - Edition 2018 の idiom 警告は有効にしておいた方がいい
  - 匿名ライフタイムを闇雲に使わない
    - 借用チェッカーとライフタイムが通っても意味的に正しいとは限らない
    - 匿名ライフタイムで十分だと明らかな場合のみ使う
      - 関数の引数と返り値が省略されたライフタイムで動くが、ライフタイムを明示したい場合
      - Trait を実装するが、メソッドでライフタイムが不要な場

---

[^1]: [Validating References with Lifetimes - The Rust Programming Language](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html) より抜粋。なお、"Call me Ishmael. Some years ago..." はハーマン・メルヴィル「白鯨」の書き出しの一節。

