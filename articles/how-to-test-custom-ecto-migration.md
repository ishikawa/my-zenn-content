---
title: "Ecto の Migration をテストする"
emoji: "🔫"
type: "tech"
topics: [elixir, ecto]
published: true
---

現在関わっている Elixir のプロジェクトでは、[Snowflake](https://blog.twitter.com/engineering/en_us/a/2010/announcing-snowflake.html) ライクなプライマリー・キーやレコードの論理削除を採用しているため、テーブル定義を簡単にするために Ecto.Migration を拡張したモジュールを使っている。

```elixir
defmodule MyApp.Repo.Migrations.CreateUsers do
  use MyApp.Migration.Resource

  @table :users

  def up do
    # 適切なプライマリー・キーを設定する
    create_resource table(@table) do
      add :name, :string, null: false
      add :email, :string, null: false

      # 論理削除のためのフラグを追加する
      resource_status(:status)
      timestamps()
    end
  end

  def down do
    # Sequence なども削除
    drop_resource(table(@table))
  end  
end
```

このモジュールのテストは、これまで `Ecto.Migration.Runner` を使って、以下のようなコードで行っていた（参考:  [Ecto 3.17 から 3.2.2 にアップデート: Migration のテスト](https://qiita.com/ishikawa@github/items/a13377036c42e05d7043)）

```elixir
defmodule MigrationModule do
  use Ecto.Migration
end

setup do
  level = Application.get_env(:logger, :level, :info)
  log = %{level: level, sql: false}

  {:ok, runner} =
    Ecto.Migration.Runner.start_link({self(), Repo, MigrationModule, :forward, :up, log})

  Ecto.Migration.Runner.metadata(runner, prefix: "test.migration.resource")
  {:ok, %{runner: runner}}
end

test "create_resource" do
  assert %Table{name: "users", primary_key: primary_key} =
           MyApp.Migration.Resource.create(table(:users), do: [])

  assert !primary_key
end
```

しかし、この実装にはいくつかの欠点がある。

1. Migration の**実装をテスト**しているため、`MyApp.Migration.Resource` の変更に弱い
2. 仕様が非公開のモジュール `Ecto.Migration.Runner` を使っているため、Ecto のバージョンアップで壊れる

1 については、ほとんど実装を変えないモジュールであるため問題にはならなかったのだが、2 については先の記事にも書いてある通り、何度か壊れている。

> もっとも、Ecto の実装が変わるたびに書き換えが必要になるのはメンテナンス性が悪い（書き換えるのも二度目）。[ecto_sql の integration test](https://github.com/elixir-ecto/ecto_sql/blob/980d67688ee41b5fd1def604b93f3dc2cd4d4548/integration_test/sql/migration.exs) のように書いた方がいいかもしれない。

今回、3.5 に上げるときも見事に壊れたので、いよいよ書き換えることにした 😥

## Migration の Integration Test

普段の開発プロセスでやっているように Migration を実行して、SQL だけを使ってテストしておけば、Ecto の実装が変わっても壊れることはない。以下の方針で書き換えることにした。

- 公開インターフェースである [Ecto.Migrator](https://hexdocs.pm/ecto_sql/Ecto.Migrator.html) で Migration を実行する
- 作成/変更されたテーブルに対して、SQL を実行して結果をテストする

[最新バージョンの ecto_sql の integration_test](https://github.com/elixir-ecto/ecto_sql/blob/980d67688ee41b5fd1def604b93f3dc2cd4d4548/integration_test/sql/migration.exs) を参考に書き直してみたので、順に説明する。

## Migration 対象モジュールの定義

まずは、テストしたい Migration を定義したモジュールを作る。なお、以下の例ではわかりやすさを優先して標準の `Ecto.Migration` を使っている。

```elixir
defmodule CreateTableMigration do
  use Ecto.Migration

  def change do
    create_resource table(:test_create_migration) do
      add :value, :integer
    end
  end
end
```

## Ecto.Adapters.SQL.Sandbox のモードを変更する

おそらく、あなたのプロジェクトのテストでは、並列テストのために [Ecto.Adapters.SQL.Sandbox](https://hexdocs.pm/ecto_sql/Ecto.Adapters.SQL.Sandbox.html) のモードを `:manual` や `:shared` にしているはずだ。残念ながら、これだと Ecto.Migrator はコネクションを取得できないので、モードを `:auto` にしてテーブルの破棄は手動で行う（Migration のロールバックもテストしたいのでこれはこれでいい）。

```elixir
setup do
  # Sets the mode of Sandbox to `:auto`. It means the repository will automatically
  # check connections out as with any other pool without transaction.
  Ecto.Adapters.SQL.Sandbox.mode(Repo, :auto)
  {:ok, migration_number: System.unique_integer([:positive]) + 1_000_000}
end
```

また、`migration_number` は、Migration のバージョンとして適当な整数を生成している。

## Migration の実行

Migration の実行は各 `describe` ブロックの `setup/2` で行うようにした。これは、Migration の up/down が確実に実行されるようにするためだ。

```elixir
describe "create_table_migration" do
  setup %{migration_number: num} do
    :ok = Ecto.Migrator.up(Repo, num, CreateTableMigration, log: false)

    on_exit(fn ->
      :ok = Ecto.Migrator.down(Repo, num, CreateTableMigration, log: false)
    end)

    :ok
  end
  ...
end
```

## SQL で結果をテスト

あとは、SQL を発行して「期待する結果」が返ってくるかをテストしていく。

```elixir
test "insert row" do
  assert {:ok, %{num_rows: 1}} =
           Repo.query("INSERT INTO test_create_migration (value) VALUES (1)")

  assert {:ok, %{rows: [[id, status]]}} =
           Repo.query("SELECT id, status FROM test_create_migration")

  assert id > 0, "id must be positive integer"
  assert status == 0, "default value: status"
end
```

データベース依存になるが、テーブル定義を調べることもできる（以下の例は PostgreSQL）。[^1] データベース依存のテストを書くかどうかはプロジェクトの要件次第だろう。

```elixir
import Ecto.Query, only: [from: 2]
...
test "column definitions" do
  columns =
    Repo.all(
      from(i in "columns",
        select: [i.column_name, i.data_type],
        where: i.table_name == ^"test_create_migration"
      ),
      prefix: "information_schema"
    )
    |> Enum.map(fn [k, v] -> {k, v} end)
    |> Map.new()

  assert columns["id"] == "bigint"
  assert columns["status"] == "smallint"
end
```

## Migration の警告

ここまでで、テスト自体はできたが、テストを実行すると以下のような警告が出てしまう。

```bash
[warn] You are running migration 1000419 but an older migration with version 20200721175245 
has already run.

This can be an issue if you have already ran 20200721175245 in production because a new
deployment may migrate 1000419 but a rollback command would revert 20200721175245 instead of
1000419.

If this can be an issue, we recommend to rollback 1000419 and change it to a version later
than 20200721175245.
```

これは、すでにアプリケーション本体の Migration でバージョン `20200721175245` などの Migration を実行しているためで、Ecto が「何か間違ってるんじゃない？」と警告を出してくれている。

この警告に対処するためには、以下のいずれかになるだろう。

1. そのまま無視する
2. `@moduletag :capture_log` でログを出さないようにする (`Ecto.Integration.MigrationTest` はこれ)
3. `Ecto.Migrator.up/4` に渡す `version` をアプリケーション本体の Migration よりも新しいものにする

今回は echo_sql に倣って 2 を選択した。ただ、他のログも捨ててしまうので、警告があまり多くなければ、1 の「そのまま放置」でもいいかもしれない。

[^1]: [PostgreSQL: Documentation: 9.5: columns](https://www.postgresql.org/docs/9.5/infoschema-columns.html)