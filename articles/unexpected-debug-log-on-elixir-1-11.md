---
title: "Elixir 1.11 の mix --no-start オプションと Logger"
emoji: "🍋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [elixir]
published: true
---

## Elixir 1.11 の Logger

Elixir 1.11 では Logger にもいくつかの改善があった。[^1]

- サポートされるログレベルが増えて[^2] Erlang/OTP 21 の新しい [`logger`](https://erlang.org/doc/man/logger.html) や Syslog の Severity [^3] と同じ種類の優先度を扱えるようになった
- 構造化データ (map と keyword list) のロギングをサポート
- [`Logger.put_module_level/2`](https://hexdocs.pm/logger/Logger.html#put_module_level/2) でモジュールごとのログレベルが設定可能に

早速、Elixir 1.11 にアップグレードして動作確認をしていたのだが、レベルを `:warn` にしていても、**テストを実行すると debug ログが出力されてしまう**、という問題に遭遇してしまった。

## テストで debug ログが出力されてしまう

Elixir 1.11 にして `mix test` を実行すると、以下のように大量のログが出力されるようになってしまった。

```sql
% mix test
...
[debug] QUERY OK db=0.5ms queue=0.1ms
INSERT INTO "users" ("display_name","email", ...
[debug] QUERY OK db=1.0ms
INSERT INTO "tokens" ("name","encrypted_token","user_id","inserted_at","updated_at") VALUES ($1,$2,$3,$4,$5) RETURNING "id" ["token7", <<204, 59, ...
INSERT INTO "organizations" ("description","display_name","status","inserted_at","updated_at") VALUES ($1,$2,$3,$4,$5,$6,$7) RETURNING "id" ["organization-755", "organization-755", ...
```

一見して、[ex_machina](https://github.com/thoughtbot/ex_machina) による fixture 生成の SQL だと分かるのだが、このログは `:debug` レベルでしか出力されないはずだ。念のため、config/test.exs を確認してみる。

```elixir:config/test.exs
config :logger,
  level: :warn,
  backends: [:console]
```

ちゃんと設定されている（ログレベルの変更については後述）[^4]

## --no-start に注意

実は、このプロジェクトでは、テストを `--no-start` オプションつきで実行するように mix.exs で設定されていた。

```elixir:mix.exs
defp aliases do
  [
    ...
    test: ["ecto.create --quiet", "ecto.migrate", "test --no-start"]
  ]
end
```

`--no-start` オプションがついていると、アプリケーション（と依存しているアプリケーション）が起動しない。`--no-start` オプションを外せば、正常にログの設定が反映されるようになった。

```bash
$ mix test
02:27:22.114 [info]  Migrations already up
............................................................................................
```

試しに、Elixir 1.10 と 1.11 で `config :logger, level: :warn ` を設定したときに `Logger.level()` がどうなるか調べてみた。

| Elixir version             | Logger.level() |
| -------------------------- | -------------- |
| Elixir 1.10.4              | `:warn`        |
| Elixir 1.10.4 (--no-start) | `:warn`        |
| Elixir 1.11.0              | `:warning`     |
| Elixir 1.11.0 (--no-start) | `:debug`       |

比較してみると、Elixir 1.11 では以下のように挙動が変わっていることが分かるだろう。

- `:warn` は `:warning` に変換される [^5]
- `--no-start` オプションをつけると設定が読み込まれない

## なぜ、--no-start で Logger の設定が反映されないのか？

まず、`--no-start` オプションが指定されているときに config.exs で指定した Logger の設定が反映されない、直接的な原因は以下のコミットである。

- [Ensure protocols are consolidated after compilation · elixir-lang/elixir@4bd3c99](https://github.com/elixir-lang/elixir/commit/4bd3c9936ba4d6d72b662655bfd4ab7c5a283e0a#diff-cffd24d00c59b8e800adffe0f0ddc267f13d12ec5f0e98c7beb8256bd73791efR60)

`--no-start` オプションが指定されていないときだけ、Logger アプリケーションを停止していることが分かると思う。

```elixir
if "--no-start" in args do
  ...
else
  # Stop Logger when starting the application as it is up to the
  # application to decide if it should be restarted or not.
  #
  # Mix should not depend directly on Logger so check that it's loaded.
  if Process.whereis(Logger) do
    Logger.App.stop()
  end
```

Elixir 1.10 では[無条件で停止されるようになっていた](https://github.com/elixir-lang/elixir/blob/1145dc01680aab7094f8a6dbd38b65185e14adb4/lib/mix/lib/mix/tasks/app.start.ex#L73)。

さて、Logger を停止しないと設定が反映されないわけだが、こうなる背景については、以下のコミットのメッセージに詳しい。[^6]

- [Ensure project's Logger config is used when running Mix tasks (#9332) · elixir-lang/elixir@78551c1](https://github.com/elixir-lang/elixir/commit/78551c11c9c97bb608e4097d2e0c36458b11f535)

つまり、Logger はアプリケーションの Mix config が準備できる前にデフォルトの設定で起動するため、アプリケーションの設定を反映させるためには再起動してやらないといけない。

## 「それでも、--no-start が必要なんだ」

この問題を解決するためには `--no-start` を取り除くのがベストな解決策だ。

そもそも、ほとんどのプロジェクトでは `--no-start` でテストを実行する必要などないのだが、今回のプロジェクトでは分散環境のテストに [whitfin/local-cluster](https://github.com/whitfin/local-cluster) を使っており、このオプションが必要だった😢

しかたがないので、mix がやっているように、アプリケーションを起動する前に Logger を停止してやる。これは、test_helper.exs に `Logger.App.stop()` を追加してやれば実現できる。

```elixir:test_helper.exs
# To avoid some potential issues with local_cluster, we have to run
# `mix test` with `--no-start` option. But it causes another issue that
# Logger can not load mix configuration. So we stop Logger here to ensure
# load mix config when the application started.
:ok = Logger.App.stop()

{:ok, _} = Application.ensure_all_started(:my_app)
```

もちろん、これは期待通りに動く。しかし、ドキュメント化されている仕様ではないので、いつまた動かなくなるかちょっと不安だ。[^7]

[^1]: [Elixir v1.11 released - The Elixir programming language](https://elixir-lang.org/blog/2020/10/06/elixir-v1-11-0-released/)
[^2]: Elixir 1.11 の Logger でサポートされるレベルの一覧 [Logger — Logger v1.11.3](https://hexdocs.pm/logger/1.11.3/Logger.html#module-levels) 
[^3]: Syslog については [RFC 3164 - The BSD Syslog Protocol](https://tools.ietf.org/html/rfc3164)
[^4]: わざわざ `backends` も上書きしているのは、本番環境で JSON ログを扱うために独自のバックエンドを使っているから
[^5]: 1.11 の CHANGELOG には "Soft-deprecations (no warnings emitted)" として `warn` ログレベルの非推奨が記載されている [elixir/CHANGELOG.md at v1.11.3 · elixir-lang/elixir](https://github.com/elixir-lang/elixir/blob/v1.11.3/CHANGELOG.md#logger-3)
[^6]: なお、このコミット自体は後ほど Revert されている [Revert "Ensure project's Logger config is used when running Mix tasks… · elixir-lang/elixir@7d57dc9](https://github.com/elixir-lang/elixir/commit/7d57dc9d9b98414652fb5c0da31e7ce9f35fe651)
[^7]: epmd を起動したり、分散環境テストのためにアプリケーションを別ノードで起動したりするので、test_helper.exs 以外で local_cluster を起動するのは難しそう