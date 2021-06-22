---
title: "あの日見た匿名ライフタイム"
emoji: "💐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: true
---

プライベートで Rust を書くようになってから 4 ヶ月ほど経ちます。

多くの初心者と同じようにライフタイムで苦しめられています。最近も**匿名ライフタイム**のことがよく分からず、いくつかプログラムを書いたりドキュメントや Web で情報を漁って調べていました。調べはじめた当初よりは理解も進んで、気をつけるポイントも分かってきたので記事にまとめておきます。

この記事では以下のことを説明していきます。

- そもそも匿名ライフタイムとは何なのか？
- 何のためにあるのか？
- 日々のプログラミングでどのように役立つのか？

けっこう長い記事なので、時間がない方は「まとめ」を読んでください。

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

もちろん、*ImportantExcerpt* 構造体の持つ `'a` というライフタイムは名前がついているので「**匿名**」ライフタイムではありません。

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

別の使われ方も見てみましょう。同じく標準ライブラリから。標準入力への排他アクセスを提供する std::io::[StdinLock](https://doc.rust-lang.org/1.52.0/std/io/struct.StdinLock.html) が std::io::[Read](https://doc.rust-lang.org/1.52.0/std/io/trait.Read.html) トレイトを実装しているコードです。[^3]

```rust
impl Read for StdinLock<'_> {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        self.inner.read(buf)
    }
```

今度は関数シグネチャではなく impl ブロックの構造体に与えるライフタイムに匿名ライフタイムが使用されています。

最後に、トレイトオブジェクトと[ライフタイム境界](https://doc.rust-lang.org/rust-by-example/scope/lifetime/lifetime_bounds.html)の組み合わせです。整数のベクタを持つ構造体に、各要素を 2 倍にするイテレータを返すメソッド `doubled_items` メソッドを実装します。[^4]

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

また、dyn Trait と impl Trait については、以下の記事が非常に参考になりました。

- [dyn Trait and impl Trait in Rust](https://www.ncameron.org/blog/dyn-trait-and-impl-trait-in-rust/)

以下の章では、これらを参照しながら、匿名ライフタイムが何のために導入されて、どう役に立つのかを紹介します。

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

しかし、ライフタイムの省略は楽なのですが、`impl` ブロックの場合は構造体に与えたライフタイムを参照できなくなってしまうので、使いどころは限られています。

たとえば、以下のコードでは `'_` が同じライフタイムを指しているように見えますが、実際にはそれぞれ新しいライフタイムが割り当てられるため期待通りには動作しません。

```rust
struct Foo<'a> {
    a: &'a i32,
}

impl Foo<'_> {
    fn a(&self) -> &'_ i32 {
        self.a
    }
}
```

構造体の実装を書く `impl` ブロックでは今後追加するメソッドでライフタイムを参照する可能性があるので、省略せずに書くのがいいでしょう。逆に、実装すべきメソッドが決まっていて、トレイト自体にライフタイムを指定する必要のない Display トレイトのようなケースではコードの見通しが良くなります。

### `dyn` トレイト / `impl` トレイトでの匿名ライフタイム

最後に、関数シグネチャに現れるトレイトにおける匿名ライフタイムについて見ていきます。

「関数シグネチャに現れるトレイト」には次の 3 種類があります。

1. 引数の位置の `impl` トレイト
2. 返り値の位置の `impl` トレイト
3. `dyn` トレイト

#### 引数の位置の `impl` トレイト

このうち引数の `impl` トレイトは[トレイト境界](https://doc.rust-lang.org/rust-by-example/generics/bounds.html)を指定するための簡便な記法 [^7] ですが、ここでは匿名ライフタイムは使えません。明示的に指定する必要があります。なので、ここではこれ以上解説しません。

#### 返り値の位置の `impl` トレイト

返り値の位置の `impl` トレイトは、関数から返す値の「実際の型」を書かずに「トレイトを実装している、ある型」を返す関数を実装することができます。これは特に、イテレータを返すときに便利です。

例えば、Vec のイテレータを返す関数を書くときこれまでは、

```rust
struct Foo {
    items: Vec<i32>,
}

impl Foo {
    fn items(&self) -> slice::Iter<i32> {
        self.items.iter()
    }
}
```

このように返り値に「実際の型」を指定する必要がありましたが、これを

```rust
fn items(&self) -> impl Iterator<Item = &i32> {
    self.items.iter()
}
```

このように具体的な型を隠蔽することができます。

また、クロージャを使ったイテレータでは具体的な型を記述することができないため、トレイトオブジェクトを Box で返す必要がありました。これも impl トレイトを使えば簡単です。

```rust
fn even_items(&self) -> impl Iterator<Item = &i32> {
    self.items.iter().filter(|x| *x % 2 == 0)
}
```

しかし、簡単にはいかないケースもあります。今度は各要素を倍にして返すイテレータを書いてみましょう。

```rust
fn doubled_items(&self) -> impl Iterator<Item = i32> {
    self.items.iter().map(|x| x * 2)
}
```

```
error[E0759]: `self` has an anonymous lifetime `'_` but it needs to satisfy a `'static` lifetime requirement
  --> ./src/main.rs:15:20
   |
14 |     fn doubled_items(&self) -> impl Iterator<Item = i32> {
   |                      ----- this data with an anonymous lifetime `'_`...
15 |         self.items.iter().map(|x| x * 2)
   |         ---------- ^^^^
   |         |
   |         ...is captured here...
   |
note: ...and is required to live as long as `'static` here
  --> ./src/main.rs:14:31
   |
14 |     fn doubled_items(&self) -> impl Iterator<Item = i32> {
   |                                ^^^^^^^^^^^^^^^^^^^^^^^^^
help: to declare that the `impl Trait` captures data from argument `self`, you can add an explicit `'_` lifetime bound
   |
14 |     fn doubled_items(&self) -> impl Iterator<Item = i32> + '_ {
   |                                                          ^^^^
```

ライフタイムのエラーが起きてしまいました。これはどういうことでしょうか？

詳しい説明は省きますが、後者の例では、[^8]

- 型パラメータに参照を含まないため、返り値のライフタイムは `'static` と推論される
- しかし、関数本体で返されている実際の値は `self` のライフタイム

このように、コンパイアによるライフタイムの推論結果が正しくないときは、明示的にライフタイムを指定してやる必要があります。そして、ここで指定すべきなのが匿名ライフタイムです。匿名ライフタイムには `&Self` 型引数のライフタイムが割り当てられています。

```rust
fn doubled_items(&self) -> impl Iterator<Item = i32> + '_ {
    self.items.iter().map(|x| x * 2)
}
```

#### `dyn` トレイト

`dyn` トレイトは[トレイト・オブジェクト](https://doc.rust-lang.org/reference/types/trait-object.html)を表すための記法です。

```rust
pub struct Screen {
    pub components: Vec<Box<dyn Clone>>,
}
```

トレイト・オブジェクトのライフタイムは[ライフタイム境界](https://doc.rust-lang.org/rust-by-example/scope/lifetime/lifetime_bounds.html)によって推論されます。ライフタイム境界が省略された場合、この記事の冒頭で解説したライフタイムの省略ルールには従わず、[*default object lifetime bound*](https://doc.rust-lang.org/reference/lifetime-elision.html#default-trait-object-lifetimes) というルールに従います。

このルールについてこれ以上深くは追求しませんが、特に推論の材料となる要素がなければ `'static` ライフタイム境界が割り当てられます（参照を含まない構造体などは `'static` ライフタイム境界で許可されるので、これはまあ妥当な判断でしょう）。

このルールによって割り当てられたライフタイムが正しくないときは、明示的にライフタイムを指定してやる必要があります...。そうです。お察しの通り、ここでも匿名ライフタイムが使えます。

```rust
fn bar(x: &i32) -> Box<dyn Debug + '_> {
    Box::new(x)
}
```

匿名ライフタイムを使うことで、通常のライフタイムの省略ルールが適用されます。つまり、上記の例だと以下のように解釈されるわけです。

```rust
fn bar<'a'>(x: &'a i32) -> Box<dyn Debug + 'a> {
    Box::new(x)
}
```

## まとめ

長々と書いてきましたが、匿名ライフタイムについてまとめます。

- **ライフタイムは[省略ルール](https://doc.rust-lang.org/reference/lifetime-elision.html)に従って省略することができる**
- **省略ルールに従ったライフタイムを明示するプレースホルダとして `'_` が使える**
- **`dyn` トレイトと `impl` トレイトのライフタイム境界で省略ルールを適用するために `'_` が使える**

省略ルールを理解し、匿名ライフタイムの使いどころを知ることでライフタイムについての理解も深まると思います。最後に、これらを活用したプログラムを書くときに役立つ設定を紹介して記事を終わります。

まず、Edition 2018 の慣習的な書き方を学ぶためにも、Rust の Lint で [rust-2018-idioms](https://doc.rust-lang.org/rustc/lints/groups.html) グループを有効にしましょう（デフォルトでは無効になっています）。rustc に `-D rust-2018-idioms` オプションを渡すか、crate の先頭で、

```rust
#![deny(rust_2018_idioms)]
```

を指定することでエラー扱いにできます。

また、ライフタイムを明示的に指定しなくても省略ルールによって省略できるケースは clippy が [needless_lifetime](https://rust-lang.github.io/rust-clippy/master/index.html#needless_lifetimes) ルールで検出してくれます。

---

[^1]: [Validating References with Lifetimes - The Rust Programming Language](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html) より抜粋。なお、"Call me Ishmael. Some years ago..." はハーマン・メルヴィル「白鯨」の書き出しの一節。
[^2]: https://github.com/rust-lang/rust/blob/1.52.1/library/std/src/io/error.rs/#L534-L547
[^3]: https://github.com/rust-lang/rust/blob/1.52.1/library/std/src/io/stdio.rs/#L416-L419
[^4]: [Returning Traits with dyn - Rust By Example](https://doc.rust-lang.org/rust-by-example/trait/dyn.html)
[^5]: [Lifetime elision - The Rust Reference](https://doc.rust-lang.org/reference/lifetime-elision.html)
[^6]: rust-2018-idioms グループに含まれています。https://doc.rust-lang.org/rustc/lints/groups.html
[^7]: ただし、全く同じではなく違いもあります。`T: Trait` で定義された関数は呼び出し側で実際の型を明示的に指定することができます (e.g. `foo::<usize>(1)`) が、`impl Trait` ではこれができません。
[^8]: [1951-expand-impl-trait - The Rust RFC Book](https://rust-lang.github.io/rfcs/1951-expand-impl-trait.html#scoping-for-type-and-lifetime-parameters) impl Trait のライフタイムについては、この RFC に書かれています。

