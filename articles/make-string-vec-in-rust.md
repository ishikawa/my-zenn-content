---
title: "Rust で String の Vec を作る"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: true
---

Rust で文字列 `&str` の [Vec](https://doc.rust-lang.org/1.52.0/std/vec/struct.Vec.html) を作るのは簡単です。

```rust
vec!["apple", "orange", "grape"]
```

しかし、ソフトウェアの設計や使っているライブラリによっては [String](https://doc.rust-lang.org/1.52.0/std/string/struct.String.html) が必要になることもあります。

```rust
vec!["apple".to_string(), "orange".to_string(), "grape".to_string()]
```

まあ、これで動くのですが、それぞれの要素が離れすぎていて、読みづらく感じます。これは [Iterator](https://doc.rust-lang.org/1.52.0/std/iter/trait.Iterator.html) と `.map()` を使えばマシにはなります。

```rust
vec!["apple", "orange", "grape"].iter().map(|s| s.to_string()).collect()
```

もう少し改良すると、最初に Vec を作る必要はありません。配列のスライスも Iterator を実装しています。

```rust
["apple", "orange", "grape"].iter().map(|s| s.to_string()).collect()
```

更に、要素の型 `String` は自明なので、`.to_string()` よりもう少し短い `.into()` が使えます。

```rust
["apple", "orange", "grape"].iter().map(|&s| s.into()).collect()
```

ただ、これは参照を外すための `&` が必要になる [^1] ので、短くはなっても少し目障りかもしれません。

## なぜ、[Into](https://doc.rust-lang.org/1.52.0/std/convert/trait.Into.html) では Auto dereference がきかないのか

str の to_string() はこのように実装されていますが、[^2]

```rust
impl ToString for str {
    #[inline]
    fn to_string(&self) -> String {
        String::from(self)
    }
}
```

Into トレイトは*すべての型*にたいして以下のように実装されています。[^3]

```rust
impl<T, U> Into<U> for T
where
    U: From<T>,
{
    fn into(self) -> U {
        U::from(self)
    }
}
```

そのため、次のコードでは、

```rust
let _: Vec<String> = ["apple", "orange", "grape"].iter().map(|s| s.into()).collect();
```

1. `s` は `&&str`
2. Into は `T` に実装されているので、`&&str` にも実装されている
3. しかし、`U` のトレイト境界 `From<T>` つまり `From<&&str>` は `String` に実装されていない

ということでコンパイルエラーになります。

```
error[E0277]: the trait bound `String: From<&&str>` is not satisfied
 --> src/main.rs:2:72
  |
2 |     let _: Vec<String> = ["apple", "orange", "grape"].iter().map(|s| s.into()).collect();
  |                                                                        ^^^^ the trait `From<&&str>` is not implemented for `String`
  |
  = help: the following implementations were found:
            <String as From<&String>>
            <String as From<&mut str>>
            <String as From<&str>>
            <String as From<Box<str>>>
          and 2 others
```

## 参考

- [rust - Why is an explicit dereference required in (*x).into(), but not in x.my_into()? - Stack Overflow](https://stackoverflow.com/questions/58082860/why-is-an-explicit-dereference-required-in-x-into-but-not-in-x-my-into)



[^1]: [Patterns - The Rust Reference](https://doc.rust-lang.org/reference/patterns.html#reference-patterns)
[^2]: https://github.com/rust-lang/rust/blob/1.52.1/library/alloc/src/string.rs/#L2285-L2290
[^3]: https://github.com/rust-lang/rust/blob/1.52.1/library/core/src/convert/mod.rs/#L533-L540
