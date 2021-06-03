---
title: "あの日見た匿名ライフタイム"
emoji: "💐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: false
---

プライベートで Rust を書くようになってから 4 ヶ月ほど経ちます。

多くの初心者と同じようにライフタイムで苦しめられています。最近も**匿名ライフタイム**のことがよく分からず、いくつかプログラムを書いたりドキュメントや Web で情報を漁って調べていました。調べはじめた当初よりは理解も進んで、気をつけるポイントも分かってきたので記事にまとめておきます。

この記事では同じような Rust 初心者向けに以下のことを説明していきます。

- そもそも匿名ライフタイムとは何なのか？
- 何のためにあるのか？
- 日々のプログラミングでどのように役立つのか？

## 匿名ライフタイムとは何か？

まずは**匿名ライフタイム**とは何でしょうか？　例として次の構造体を見てください。

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

別の例として、標準ライブラリの std::io::error の [Error](https://doc.rust-lang.org/1.52.0/std/io/struct.Error.html) 構造体が [Display](https://doc.rust-lang.org/std/fmt/trait.Display.html) トレイトを実装するコードを見てみましょう。[^2]

```rust
impl fmt::Display for Error {
    fn fmt(&self, fmt: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self.repr {
            Repr::Os(code) => {
                let detail = sys::os::error_string(code);
                write!(fmt, "{} (os error {})", detail, code)
            }
            Repr::Custom(ref c) => c.error.fmt(fmt),
            Repr::Simple(kind) => write!(fmt, "{}", kind.as_str()),
        }
    }
}
```

std::fmt::[Formatter](https://doc.rust-lang.org/std/fmt/struct.Formatter.html) に指定するライフタイムに、匿名ライフタイムが使われていることが分かります。これは関数の返り値ではなく引数で使われている点を除けば、最初の例と同じ使われ方ですね。

では、別の使われ方も見てみましょう。同じく標準ライブラリから。標準入力への排他アクセスを提供する std::io::[StdinLock](https://doc.rust-lang.org/1.52.0/std/io/struct.StdinLock.html) が std::io::[Read](https://doc.rust-lang.org/1.52.0/std/io/trait.Read.html) トレイトを実装しているコードです。[^3]

```rust
impl Read for StdinLock<'_> {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        self.inner.read(buf)
    }
```

今度は関数シグネチャではなく impl ブロックの構造体に与えるライフタイムに匿名ライフタイムが使用されています。

最後に、トレイトオブジェクトとライフタイム境界の組み合わせです。整数のベクタを持つ構造体に、各要素を 2 倍にするイテレータを返すメソッド `double_items` を実装します。[^4]

```rust
struct Foo {
    items: Vec<i32>,
}

impl Foo {
    fn doubled_items(&self) -> Box<dyn Iterator<Item = i32> + '_> {
        Box::new(self.items.iter().map(|x| x * 2))
    }
}
```

この例では、Box を使わなくても [impl Trait](https://doc.rust-lang.org/rust-by-example/trait/impl_trait.html) で書き直すことができます。

```rust
fn doubled_items(&self) -> impl Iterator<Item = i32> + '_ {
    Box::new(self.items.iter().map(|x| x * 2))
}
```

どちらの例でもライフタイム境界に匿名ライフタイムが使われていることがわかります。

### Edition 2018

匿名ライフタイム（および、先程の例でも紹介した [dyn Trait](https://doc.rust-lang.org/edition-guide/rust-2018/trait-system/dyn-trait-for-trait-objects.html) と [impl Trait](https://doc.rust-lang.org/edition-guide/rust-2018/trait-system/impl-trait-for-returning-complex-types-with-ease.html)）は Rust Edition 2018 で導入されました。この記事も Edition 2018 の [Edition Guide](https://doc.rust-lang.org/edition-guide/rust-2018/) と、関連する RFC を参照しています。

- ['_, the anonymous lifetime - The Edition Guide](https://doc.rust-lang.org/edition-guide/rust-2018/ownership-and-lifetimes/the-anonymous-lifetime.html)
- [Lifetime elision in impl - The Edition Guide](https://doc.rust-lang.org/edition-guide/rust-2018/ownership-and-lifetimes/lifetime-elision-in-impl.html)
- [1951-expand-impl-trait - The Rust RFC Book](https://rust-lang.github.io/rfcs/1951-expand-impl-trait.html#scoping-for-type-and-lifetime-parameters)

以下の章では、これらの文献を参照しながら、匿名ライフタイムが何のために導入されて、どう役に立つのかを紹介します。

## 匿名ライフタイムの意義と効用

この章では先程の例で見てもらった匿名ライフタイムの使い方それぞれについて、意義と効用について書いていきます。

- 関数での匿名ライフタイム
- `impl` ブロックでの匿名ライフタイム
- `dyn Trait` と `impl Trait` での匿名ライフタイム

### 関数での匿名ライフタイム

関数の匿名ライフタイムを理解するためには、ライフタイムの省略ルールについて理解する必要があります。では、ここでライフタイムの省略ルールをおさらいしておきましょう。[^5]

Rust の参照には必ずライフタイムが割り当てられますが、関数やクロージャでよくあるパターンではその指定を省略できるようになっています。

1. 引数に対して省略したライフタイムは、それぞれ別のライフタイムになります。
2. 引数のライフタイムがひとつしかない場合は、そのライフタイムが返り値で省略されたライフタイムに適用されます。
3. メソッドの引数に複数のライフタイムがあって、そのうちひとつが `&Self` 型か `&mut Self` 型である場合、そのライフタイムが返り値で省略されたライフタイムに適用されます。

つまり、以下のようなプログラムでは、

```rust
struct Foo {
    items: Vec<i32>,
}

impl Foo {
    fn items(&self) -> std::slice::Iter<i32> {
        self.items.iter()
    }
}
```

構造体 Foo の `items` メソッドで省略されたライフタイムを明示すると以下のようになります。

```rust
fn items<'a'>(&'a self) -> std::slice::Iter<'a, i32> {
    self.items.iter()
}
```

std::slice::[Iter](https://doc.rust-lang.org/1.52.0/std/slice/struct.Iter.html#impl-DoubleEndedIterator) はひとつのライフタイムを受け取るはずですが、上記の省略ルール 3 により省略されていたわけです。

似たような例として、今度は返り値のライフタイムが省略される例を見てみましょう。

```rust
struct StrWrapper<'a> {
    string: &'a str,
}

fn make_wrapper(string: &str) -> StrWrapper {
    StrWrapper { string }
}

fn main() {
    let w = make_wrapper("hello");
    println!("string = {}", w.string); // string = hello
}
```

`make_wrapper` 関数の返り値は参照を含む構造体 `StrWrapper` を返しますが、そのライフタイム `'a` は省略ルール 2 によって省略することができ、問題なくコンパイルできます。

```sh
$ rustc ./src/main.rs
$ ./main
string = hello
```

しかし、ここで [elided_lifetimes_in_paths](https://doc.rust-lang.org/rustc/lints/listing/allowed-by-default.html#elided-lifetimes-in-paths) ルールを有効にしてコンパイルしてみましょう。[^6]

```sh
$ rustc -D elided_lifetimes_in_paths ./src/main.rs
error: hidden lifetime parameters in types are deprecated
 --> ./src/main.rs:5:34
  |
5 | fn make_wrapper(string: &str) -> StrWrapper {
  |                                  ^^^^^^^^^^- help: indicate the anonymous lifetime: `<'_>`
  |
  = note: requested on the command line with `-D elided-lifetimes-in-paths`
```

型のライフタイムパラメータが省略されていることでエラーになってしまいました。参照を含む構造体のライフタイムパラメータの省略は Rust 2018 から非推奨となりました。ライフタイムパラメータが省略されてしまうと、構造体が参照を借用していることが分かりづらくなるためです（上記 [Lint ルール](https://doc.rust-lang.org/rustc/lints/listing/allowed-by-default.html#elided-lifetimes-in-paths)に詳しい説明があります）。

では、この警告を直すにはどうしたらいいでしょうか？　もちろん、ライフタイムを明示的に指定すれば大丈夫ですが、これはやはり面倒ですね。

```rust
fn make_wrapper<'a>(string: &'a str) -> StrWrapper<'a> {
    StrWrapper { string }
}
```

そこで Edition 2018 では `'_` プレースホルダによって、ライフタイムが省略されていることを明示することができるようになりました。

```rust
fn make_wrapper(string: &str) -> StrWrapper<'_> {
    StrWrapper { string }
}
```

- `'_` プレースホルダによる**匿名ライフタイム**は宣言する必要がありません
- 詳細は ['_, the anonymous lifetime - The Edition Guide](https://doc.rust-lang.org/edition-guide/rust-2018/ownership-and-lifetimes/the-anonymous-lifetime.html) を参照してください

### `impl` ブロックでの匿名ライフタイム

また、impl ブロックでのライフタイムでも匿名ライフタイムを使うことができます。たとえば、先の例の StrWrapper で std::fmt::[Display](https://doc.rust-lang.org/1.52.0/std/fmt/trait.Display.html) トレイトを実装することを考えてみましょう。

```rust
impl<'a> Display for StrWrapper<'a> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        writeln!(f, "string = {}", self.string)
    }
}

fn main() {
    let w = make_wrapper("hello");
    println!("{}", w); // string = hello
}
```

この `impl<'a> Display for StrWrapper<'a>` のライフタイムも省略して、構造体のライフタイムパラメータは匿名ライフタイムを使って明示することができます。

```rust
impl Display for StrWrapper<'_> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        writeln!(f, "string = {}", self.string)
    }
}
```

しかし、ライフタイムの省略は楽なのですが、`impl` ブロックの場合は構造体に与えたライフタイムを参照できなくなってしまうので、使いどころは限られています。構造体の実装を書く `impl` ブロックでは今後追加するメソッドでライフタイムを参照する可能性があるので、省略せずに書くのがいいでしょう。逆に、実装すべきメソッドが決まっていて、トレイト自体にライフタイムを指定する必要のない Display トレイトのようなケースではコードの見通しも良くなり、非常に役立ちます。





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
[^2]: https://github.com/rust-lang/rust/blob/1.52.1/library/std/src/io/error.rs/#L534-L547
[^3]: https://github.com/rust-lang/rust/blob/1.52.1/library/std/src/io/stdio.rs/#L416-L419
[^4]: [Returning Traits with dyn - Rust By Example](https://doc.rust-lang.org/rust-by-example/trait/dyn.html)
[^5]: [Lifetime elision - The Rust Reference](https://doc.rust-lang.org/reference/lifetime-elision.html)
[^6]: rust-2018-idioms グループに含まれています。https://doc.rust-lang.org/rustc/lints/groups.html
