---
title: "Clap の Derive API で列挙型のコマンドラインオプションを実装する"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: true
---

[Clap v3](https://github.com/clap-rs/clap) の [Derive API](https://github.com/clap-rs/clap/blob/v3.1.9/examples/tutorial_derive/README.md) と [ArgEnum](https://docs.rs/clap/3.1.9/clap/trait.ArgEnum.html) を使うと、列挙型 (Enum) のコマンドラインオプションを簡単に実装することができる（[ソースコード](https://github.com/ishikawa/example-rust-clap-derive-api-arg-enum/blob/main/src/bin/example-1.rs)）。

```rust
use clap::Parser;

#[derive(Debug, Clone, clap::ArgEnum)]
enum Level {
    Debug,
    Info,
    Warning,
    Error,
}

#[derive(clap::Parser)]
struct Args {
    #[clap(arg_enum, long = "level")]
    level: Level,
}

fn main() {
    let args = Args::parse();
    println!("level = {:?}", args.level);
}
```

このプログラムを実行すると、`level` オプションを受けつけ、取りうる値も正しく出力できることが分かる。

```
$ cargo run --bin example-1 -- -h
    Finished dev [unoptimized + debuginfo] target(s) in 0.02s
     Running `target/debug/example-1 -h`
example-rust-clap-compound-arg-enum 

USAGE:
    example-1 --level <LEVEL>

OPTIONS:
    -h, --help             Print help information
        --level <LEVEL>    [possible values: debug, info, warning, error]
```

しかし、`clap::ArgEnum` を derive する方法には制限もあり、Enum の要素に構造体を持たせたりすることはできない。

## コマンドラインオプションの Enum にデータを持たせる

たとえば上記の例で、コマンドラインオプションで `--level error` なら `Level::Error.panic=false`, `--level error-panic` なら `Level::Error.panic=true` にしたいとしよう。まずは、`Level::Error` に `panic` フィールドを追加してみる（[ソースコード](https://github.com/ishikawa/example-rust-clap-derive-api-arg-enum/blob/main/src/bin/example-2.rs)）

```rust
#[derive(Debug, Clone, clap::ArgEnum)]
enum Level {
    Debug,
    Info,
    Warning,
    Error { panic: bool },
}
```

しかし、これはコンパイルできず、エラーになる。

```
error: `#[derive(ArgEnum)]` only supports non-unit variants, unless they are skipped
 --> src/bin/example-1.rs:8:5
  |
8 |     Error { panic: bool },
  |     ^^^^^
```

この場合、derive マクロで `clap::ArgEnum` を実装できないので、自前で実装する必要がある（[ソースコード](https://github.com/ishikawa/example-rust-clap-derive-api-arg-enum/blob/main/src/bin/example-3.rs)）。

```rust
const LEVEL_VALUE_VARIANTS: [Level; 5] = [
    Level::Debug,
    Level::Info,
    Level::Warning,
    Level::Error { panic: false },
    Level::Error { panic: true },
];

impl clap::ArgEnum for Level {
    fn value_variants<'a>() -> &'a [Self] {
        &LEVEL_VALUE_VARIANTS
    }

    fn to_possible_value<'a>(&self) -> Option<clap::PossibleValue<'a>> {
        let name = match self {
            Level::Debug => "debug",
            Level::Info => "info",
            Level::Warning => "warning",
            Level::Error { panic } => {
                if *panic {
                    "error-panic"
                } else {
                    "error"
                }
            }
        };

        Some(clap::PossibleValue::new(name))
    }
}
```

コマンドラインオプションで取りうる Level の値を `LEVEL_VARIANTS` で配列定数として定義し、それを使って ArgEnum を実装した。

ただ、この実装では「取りうる Level の値」が `clap::ArgEnum::to_possible_value` に実装でも定義されてしまっている。もう一手間かけて、**データ駆動のプログラム**にすれば、この重複を取り除くことができる。

## データ駆動で実装の重複を取り除く

まずは、`Level` に `Copy` と `PartialEq` (`Eq`) トレイトを実装する（[ソースコード](https://github.com/ishikawa/example-rust-clap-derive-api-arg-enum/blob/main/src/bin/example-4.rs)）。

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
enum Level {
  ...
}
```

次に、コマンドラインオプションに指定できる文字列と `Level` の組をタプルにし、これを配列定数で定義する。

```rust
const LEVEL_NAME_TO_VALUE_VARIANTS: [(&'static str, Level); 5] = [
    ("debug", Level::Debug),
    ("info", Level::Info),
    ("warning", Level::Warning),
    ("error", Level::Error { panic: false }),
    ("error-panic", Level::Error { panic: true }),
];
```

そして、**この配列の要素を使って、Level だけを要素に含む配列を作り**、[`clap::ArgEnum::value_variants`](https://docs.rs/clap/3.1.9/clap/trait.ArgEnum.html#tymethod.value_variants) の実装で使うようにする。

```rust
const LEVEL_VALUE_VARIANTS: [Level; LEVEL_NAME_TO_VALUE_VARIANTS.len()] = [
    LEVEL_NAME_TO_VALUE_VARIANTS[0].1,
    LEVEL_NAME_TO_VALUE_VARIANTS[1].1,
    LEVEL_NAME_TO_VALUE_VARIANTS[2].1,
    LEVEL_NAME_TO_VALUE_VARIANTS[3].1,
    LEVEL_NAME_TO_VALUE_VARIANTS[4].1,
];

impl clap::ArgEnum for Level {
    fn value_variants<'a>() -> &'a [Self] {
        &LEVEL_VALUE_VARIANTS
    }
  	...
}
```

残念ながら、[`array::map`](https://doc.rust-lang.org/std/primitive.array.html#method.map) は [*const fn*](https://doc.rust-lang.org/reference/const_eval.html#const-functions) ではないため、このようにすべての要素を列挙する必要がある。[^1] 幸い、配列の長さ指定には別の配列の長さを使えるので、`LEVEL_NAME_TO_VALUE_VARIANTS` に新しい要素を追加したときはコンパイラが教えてくれる。

同様にして、[`clap::ArgEnum::to_possible_value`](https://docs.rs/clap/3.1.9/clap/trait.ArgEnum.html#tymethod.to_possible_value) も実装できる。

```rust
fn to_possible_value<'a>(&self) -> Option<clap::PossibleValue<'a>> {
    LEVEL_NAME_TO_VALUE_VARIANTS
        .iter()
        .find(|(_, level)| self == level)
        .map(|(name, _)| clap::PossibleValue::new(name))
}
```

`clap::ArgEnum` には他にも [`clap::ArgEnum::from_str`](https://docs.rs/clap/3.1.9/clap/trait.ArgEnum.html#method.from_str) があるが、こちらは[デフォルト実装](https://docs.rs/clap/3.1.9/src/clap/derive.rs.html#425-435)が提供されているので改めて実装する必要はない。

ただ、デフォルト実装のコードを読めば分かるが、上記の `clap::ArgEnum::to_possible_value` の実装と合わせると、$n^2$ のアルゴリズムになってしまう。とはいえ、これが問題になるほど、オプションの個数が多くなることはないだろう。



[^1]: [Will array's map function be const at some point? - Rust Internals](https://internals.rust-lang.org/t/will-arrays-map-function-be-const-at-some-point/16419)
