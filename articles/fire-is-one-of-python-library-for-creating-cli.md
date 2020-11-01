---
title: "Python で簡単にコマンドラインツールを作るなら Fire が便利"
emoji: "🔥"
type: "tech"
topics: ["python"]
published: true
---

**とにかく簡単にコマンドラインツールを作りたい。** Google 製のオープンソースライブラリ [Fire](https://google.github.io/python-fire/) は、そんなあなたのためのライブラリだ。

1. 使い方はこれ以上ないほどシンプル。`fire.Fire()` 関数を呼ぶだけ
2. インタラクティブな開発やデバッグをサポートする機能
3. ライブラリの動作確認にも便利

## 簡単にコマンドラインツールが作れる

Python でコマンドラインツールを開発するためのライブラリといえば [click](https://click.palletsprojects.com/) が有名だろう。

では、サンプルとして「指定された回数だけ指定された相手に挨拶する」コマンドラインツールを書いてみよう。このツールは以下のように実行でき、`--count` オプションが省略された場合はデフォルトとして `1` が使われるものとする。

```bash
$ python ./hello_click.py --name World --count 5
Hello World!
Hello World!
Hello World!
Hello World!
Hello World!
```

click を使って実装すると以下のようになる（click のホームページに掲載されているサンプルを参考にした）。

```python
import click

@click.command()
@click.option("--name", required=True)
@click.option("--count", default=1)
def hello(name, count):
    for _ in range(count):
        print("Hello %s!" % name)

if __name__ == "__main__":
    hello()
```

一方、Fire を使って実装すればもっと簡単だ。

```python
import fire

def hello(name, count=1):
    for _ in range(count):
        print("Hello %s!" % name)

if __name__ == "__main__":
    fire.Fire(hello)
```

コマンドラインツール化したい関数やクラスを引数にして `fire.Fire()` を呼ぶだけというシンプルさ。

:::message

もちろん、click にはコマンドラインツールの実装に必要となるさまざまな機能が提供されており、コードの短さだけをもって「click より Fire を使うべき」とは言えません。この記事の最後に「Fire と click の使い分け」として見解を述べています。

:::

`fire.Fire()` の詳しい使い方については、[公式のガイド](https://google.github.io/python-fire/guide/)を参照してほしい。後述するが、コード中で `fire.Fire()` を呼ばずに、Python インタプリタの `-m` オプションで Fire をロードするだけでも使うことができる。

## インタラクティブな開発をサポート

また、重要な特徴として、click のデコレータを用いたアプローチと異なり、Fire は対象となる関数やクラスのインターフェースを変更しないという点があげられる。これはつまり、**実装中のコードがそのままテストや REPL (対話型シェル) で使える**、ということであり、トライ＆エラーを繰り返すインタラクティブな開発では重要な特徴だ。

さきほどの click の例だと、`hello()` 関数は `@click.command()` デコレータでラップされているため、そのまま使うことはできない。

```python
$ python
Python 3.6.6 (default, Mar 23 2020, 11:51:32)
...
>>> import hello_click
>>> hello_click.hello
<Command hello>
>>> hello_click.hello(name='World')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/Users/takanori_is/Developer/Workspace/Zenn/workspace/fire-is-one-of-python-library-for-creating-cli/.venv/lib/python3.6/site-packages/click/core.py", line 829, in __call__
    return self.main(*args, **kwargs)
  File "/Users/takanori_is/Developer/Workspace/Zenn/workspace/fire-is-one-of-python-library-for-creating-cli/.venv/lib/python3.6/site-packages/click/core.py", line 781, in main
    with self.make_context(prog_name, args, **extra) as ctx:
  File "/Users/takanori_is/Developer/Workspace/Zenn/workspace/fire-is-one-of-python-library-for-creating-cli/.venv/lib/python3.6/site-packages/click/core.py", line 698, in make_context
    ctx = Context(self, info_name=info_name, parent=parent, **extra)
TypeError: __init__() got an unexpected keyword argument 'name'
```

関数に `name` 引数を与えて動作確認しようとしてもエラーになってしまう。

一方、Fire の場合はそのまま REPL でも呼び出すことができる。さらに、スクリプトを実行するときに `--interactive` オプションを指定することで、直接 REPL に入ることができる（今回のように、Fire 自体にオプションを渡すときは `--` で区切る）。

```python
$ python ./hello_fire.py -- --interactive
Fire is starting a Python REPL with the following objects:
Modules: fire
Objects: component, hello, hello_fire.py, result, trace

Python 3.6.6 (default, Mar 23 2020, 11:51:32)
>>> hello
<function hello at 0x103f81ea0>
>>> hello('World')
Hello World!
```

実行例を見て分かる通り、`--interactive` オプションで起動した REPL 環境では、必要なモジュールが自動でインポートされているので、**毎回 `import` を入力する必要がない**。これだけで開発中のストレスがだいぶ減る。

## ライブラリの動作確認にも便利

すこし前にも書いたが、Fire は Python インタプリタの `-m` でロードするだけでも使うことができる。これと [Fire がサポートするさまざまな呼び出し方法](https://google.github.io/python-fire/using-cli/)を組み合わせると、既存ライブラリの動作確認も簡単にできてしまう。

たとえば、画像処理ライブラリとして有名な [Pillow](https://pillow.readthedocs.io/en/stable/) を試してみたい、としよう。

[公式のチュートリアル](https://pillow.readthedocs.io/en/stable/handbook/tutorial.html)も充実しているので、Python の REPL で試すこともできるが、画像ファイルのパスを手入力したり、変換結果を確認するために REPL とシェルと行き来するのも多少面倒だ。

Fire を使うと、コマンドラインから動作確認ができる。

```bash
$ python -m fire PIL.Image open ./cat.jpeg - format
JPEG
```

これは以下のコードとほぼ等価である。

```python
import PIL

im = PIL.Image.open('./cat.jpeg')
print(im.format)
```

また、適宜 `--help` を表示すれば、呼び出せるメソッドやプロパティが一覧されるので便利だ。

```bash
$ python -m fire PIL.Image open ./cat.jpeg --help
NAME
    PIL.Image open ./cat.jpeg

SYNOPSIS
    PIL.Image open ./cat.jpeg - GROUP | COMMAND | VALUE
...
COMMANDS
    COMMAND is one of the following:

     alpha_composite
       'In-place' analog of Image.alpha_composite. Composites an image onto this image.

     close
       Closes the file pointer, if possible.
...
VALUES
    VALUE is one of the following:

     bits

     category

     custom_mimetype
...
```

## Fire と click の使い分け

今回、Fire を試してみて「開発中に活躍するライブラリだな」と感じた。

書き捨てのスクリプトであれば Fire のまま使いつづけてもいいが、エンドユーザー向けに提供するコマンドラインツールであれば、最終的には（パラメータのチェックやヘルプ実装が柔軟な） click で実装するのがよさそうだ。

1. まずは実現したいタスクを処理する関数/クラスを実装する
2. Fire で開発者向けのコマンドラインインターフェースを用意して、インタラクティブな開発を進める
3. 当面の動作確認が終わったら、1 の関数/クラスを対象に自動テストを作り、テスト駆動開発に移行する
4. 最後に、1 の関数/クラスをラップする形でエンドユーザー向けのコマンドラインインターフェースを click で実装する

もしも、ぼくが今後、エンドユーザー向けにコマンドラインツールを開発するなら、このような手順で進めるだろう。