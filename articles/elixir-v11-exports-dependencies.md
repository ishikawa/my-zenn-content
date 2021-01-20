---
title: "Elixir 1.11 の Exports dependencies について調べた"
emoji: "🤠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [elixir]
published: true
---

昨年、2020 年の 10 月に [Elixir 1.11 がリリースされた](https://elixir-lang.org/blog/2020/10/06/elixir-v1-11-0-released/)が、コンパイラの改善のひとつである *Exports dependencies* について動作も含めて調べた。

## Exports dependencies で改善するところ

Elixir 1.10 までは `import` や `require` したモジュールは *Compile time dependencies* となり、依存先のモジュール（のファイル）が変更されると、依存しているモジュールも再コンパイルする必要があった。

Elixir 1.11 では、モジュールの公開インターフェース（`struct` と関数）のみを使う場合は *Exports dependencies* となり、公開インターフェースが変わらない限り再コンパイルの必要はなくなった。[^1]

これによって、ソースコードを変更したときに再コンパイルが必要なファイルが減り「修正 - ビルド - 実行」のサイクルが短くなる。ビルドに必要なコンパイル時間は開発者の生産性に直結する重要な指標なので、これは嬉しい。

## Exports dependencies を試してみよう

具体例を見た方が分かりやすいと思うので、以下のふたつのモジュールを考える。[^2]

```elixir:moduleA.ex
defmodule ElixirV11.ModuleA do
  alias ElixirV11.ModuleB

  import ModuleB, only: [fetch_name: 1]

  def hello(%ModuleB{} = m) do
    case fetch_name(m) do
      {:ok, name} ->
        IO.puts("Hello, #{name}!")

      :error ->
        IO.puts("Who?")
    end
  end
end
```

```elixir:moduleB.ex
defmodule ElixirV11.ModuleB do
  defstruct name: nil

  def new(name), do: %__MODULE__{name: name}

  def fetch_name(%__MODULE__{name: nil}), do: :error
  def fetch_name(%__MODULE__{name: name}), do: {:ok, name}
end
```

ModuleA は ModuleB の struct と fetch_name/1 関数に依存している。このふたつは公開インターフェースなので、`moduleA.ex` の `import ModuleB, only: [fetch_name: 1]` は、Elixir 1.10 では *Compile time dependencies* だったが、Elixir 1.11 では *Exports dependencies* になっている。

そのため、`moduleB.ex` を変更したときに、Elixir 1.10 では `moduleA.ex` の再コンパイルが必要だが、Elixir 1.11 では不要になった。

```bash
# Elixir 1.10
$ touch lib/moduleB.ex    
$ mix compile --verbose
Compiling 1 file (.ex)
Compiled lib/moduleB.ex
Compiled lib/moduleA.ex

# Elixir 1.11
$ touch lib/moduleB.ex 
$ mix compile --verbose
Compiling 1 file (.ex)
Compiled lib/moduleB.ex
```

`touch` だけでは効果が分かりづらいので、公開インターフェースを変えない範囲で `ModuleB` の実装を変えてみよう。

```elixir:moduleB.ex
defmodule ElixirV11.ModuleB do
  defstruct name: 1

  def new(name), do: %__MODULE__{name: check_name!(name)}

  def fetch_name(%__MODULE__{name: nil}), do: :error
  def fetch_name(%__MODULE__{name: name}), do: {:ok, name}

  defp check_name!(%__MODULE__{name: name}) when is_binary(name), do: name
end
```

`check_name!/1` というプライベートな関数を追加し、`new/1` 内で使うように変えた。これは公開インターフェースに変更がないので、これも再コンパイルは不要なはずだ。

```bash
# Elixir 1.10
$ mix compile --verbose
Compiling 1 file (.ex)
Compiled lib/moduleB.ex
Compiled lib/moduleA.ex

# Elixir 1.11
$ mix compile --verbose
Compiling 1 file (.ex)
Compiled lib/moduleB.ex
```

目論見通り、これも Elixir 1.11 ではコンパイル不要になっている。

## 実プロジェクトでの効果

この変更が加えられた [Git コミット](https://github.com/elixir-lang/elixir/commit/9a6db666a107b42c1242cf727eafe7638674f3cc)のメッセージにも

> making imports more feasible for large projects.

と書かれている通り、大規模プロジェクトでも `import/2` を気軽に使えるようになりそうだ。実際に、今関わっているプロジェクトでも、Repo を操作するモジュール（さまざまな箇所で import されている）を変更して再コンパイルされるファイル数を調べたところ、

- **Elixir 1.10** - 48
- **Elixir 1.11** - 14

と激減していた。

[^1]: ただし、マクロを使っている場合は Compile time dependencies となる。このへんは [mix xref](https://hexdocs.pm/mix/Mix.Tasks.Xref.html) のページの **Dependencies types** に詳しく説明されている。
[^2]: なお、該当するコンパイラへの変更は多分これ [Do not make requires/imports compile-time dependencies · elixir-lang/elixir@9a6db66](https://github.com/elixir-lang/elixir/commit/9a6db666a107b42c1242cf727eafe7638674f3cc)