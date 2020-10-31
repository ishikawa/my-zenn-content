---
title: "Poetry の virtualenv を VSCode に認識させる"
emoji: "🐤"
type: "tech"
topics: ["python", "vscode", "poetry"]
published: false
---

## この記事で分かること

- Visual Studio Code (以下、VSCode) の Python 拡張が [Python 仮想環境](https://www.python.org/dev/peps/pep-0405/) (以下、virtualenv) を認識する条件
- [Poetry](https://python-poetry.org/) で virtualenv をプロジェクトのディレクトリ直下に作る方法

## この記事を書いた動機と結論

Python パッケージの依存関係を Poetry で管理しているプロジェクトで、エディタとして VSCode を使っている。しかし、Poetry でインストールしたパッケージを、コード中で `import` するとエラーになるので、その原因と対策を調べた記録である。

先に結論から書く。**以下のコマンドを実行することで、virtualenv がプロジェクトのディレクトリ直下に作られるようになり、VSCode でも認識されるようになる。**

```bash
$ poetry --version
Poetry version 1.0.5
$ poetry config virtualenvs.in-project true --local
$ poetry env remove 3.6  # 既存の virtualenv を削除。指定する Python バージョンは適宜調整
$ poetry install
```

## VSCode の Python 拡張が virtualenv を認識する条件

まずは、VSCode がどのようにして、プロジェクトで使用される virtualenv を見つけるのか調べてみる。[公式ドキュメント](https://code.visualstudio.com/docs/python/environments#_where-the-extension-looks-for-environments)によると、主に以下の場所にある virtualenv を認識するようだ。

1. ワークスペースまたはプロジェクトのディレクトリ直下
2. `python.venvPath` で指定されたディレクトリ以下に配置された virtualenv
   - [General Settings](https://code.visualstudio.com/docs/python/settings-reference#_general-settings) には `python.venvPath` の記載がなく、`python.venvFolders` のみ

Poetry はデフォルトの設定だと virtualenv をプロジェクトのディレクトリ直下ではなくキャッシュディレクトリ (Mac だと `~/Library/Caches/pypoetry`) に作ってしまう。

Poetry のキャッシュディレクトリを `python.venvPath` に設定することでも解決できそうだが、このディレクトリには複数の virtualenv が配置されることと、**Git 管理していないエディタの設定にあまり依存したくない**ので、今回は以下の方法を採用する。

- Poetry の設定を変更して、プロジェクトのディレクトリ直下に virtualenv を作る
- Poetry の設定ファイルを Git 管理する

## プロジェクトのディレクトリ直下に virtualenv を作る

Poetry の設定は以下のコマンドで確認することができる。

```bash
$ poetry config --list
cache-dir = "..."
virtualenvs.create = true
virtualenvs.in-project = false
virtualenvs.path = "{cache-dir}/virtualenvs"  # ...
```

これらのうち、[`virtualenvs.in-project`](https://python-poetry.org/docs/configuration/#virtualenvsin-project-boolean) が `false` になっていることが、キャッシュディレクトリに virtualenv が作られる原因である。この設定を変更するには、以下のコマンドを実行する。

```bash
$ poetry config virtualenvs.in-project true --local
```

また、`--local` フラグをつけることで、poetry.toml というファイルに設定が記録され、Git 管理できるようになる（参考: [Local configuration](https://python-poetry.org/docs/configuration/#local-configuration)）。

```toml
$ cat poetry.toml
[virtualenvs]
in-project = true
```

最後に、既存の virtualenv を削除して install しなおせば、プロジェクトのディレクトリ直下に `.venv` が作られているはずだ。

```bash
$ poetry env remove 3.6  # 既存の virtualenv を削除。指定する Python バージョンは適宜調整
$ poetry install
```

