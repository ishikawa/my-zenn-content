---
title: "Cypress をサーバ起動まで待機させる"
emoji: "🚀"
type: "tech"
topics: ["javascript", "e2e", "test"]
published: true
---

サーバの起動を待ってからフロントエンドのテストを実行したければ、[bahmutov/start-server-and-test](https://github.com/bahmutov/start-server-and-test) が使えるよ。

…ということを書き残しておきたかった。記事の残りは背景説明と簡単なセットアップ方法の紹介なので、急いでいる人は読む必要はない。とりあえず、上記の start-server-and-test を導入して、

```bash
$ npm install --save-dev start-server-and-test
```

こんなふうに使う。

```bash
$ npx start-server-and-test 'http-server -c-1 .' http://localhost:3333 'cypress run'
```

ただし、web pack-dev-server を使っている場合は、URL を `http-get://` で指定する（後述）。

## 背景 - 不安定な E2E テスト

フロントエンドの E2E テストに [cypress](https://www.cypress.io/) を使っている。テストは [CircleCI](https://circleci.com/) で実行しているのだが、このテストが最近、E2E だけ頻繁に失敗するようになってしまった。再実行しても成功したりしなかったり…。いわゆる「不安定なテスト (Flaky Tests)」[^1] であり、地味に開発者をイライラさせる嫌なやつである。

### 原因究明

しばらくは放置していたのだが、さすがに我慢の限界に達して調査してみることにした。E2E テストは CircleCI ワークフローのジョブとして、以下のように実装されていた。

1. サーバを起動するステップを `background` オプションつきでバックグラウンド実行
2. そのあとのステップで `cypress run` を実行

失敗したときのログを追ってみると、1 のサーバ起動が JavaScript コードのトランスパイルとバンドルで時間がかかり、2 でのサーバ接続確認がタイムアウトしているようだった。

```
Opening Cypress...
Cypress could not verify that this server is running:

  > http://localhost:8080

We are verifying this server because it has been configured as your `baseUrl`.

Cypress automatically waits until your server is accessible before running tests.

We will try connecting to it 3 more times...
We will try connecting to it 2 more times...
We will try connecting to it 1 more time...

Cypress failed to verify that your server is running.

Please start this server and then run Cypress again.
```

3 回リトライしているのが見てとれるが、それでも足りなかったようだ。

### 起動時のリトライ回数は増やせない...

当初は「接続確認のリトライ回数を増やせばいいだろう」と考えていたのだが、残念ながらそれはできないらしい[^2]（[失敗したテストのリトライ](https://docs.cypress.io/guides/guides/test-retries.html)はできる）。

というわけで、件の Issue でも紹介されている [bahmutov/start-server-and-test](https://github.com/bahmutov/start-server-and-test) を試してみることにした。この npm モジュールを使うことで、

1. サーバが起動し、指定されたエンドポイントに接続できるまで待機
2. 接続が確認できたらテストを実行

することができる（なので、サーバを起動するステップと cypress を実行するステップはひとつにまとめられる）。

## start-server-and-test のセットアップ

導入は npm から。開発時にしか使わないので devDependencies として追加する。

```bash
$ npm install --save-dev start-server-and-test
```

追加すると、`start-server-and-test` というコマンドが使えるようになる。このコマンドの基本的な使い方はこんな感じ。

```bash
$ start-server-and-test <SERVER-COMMAND> <URL or PORT> <TEST-COMMAND>
```

README には npm script を使う方法が、まず紹介されている。

```json
{
    "scripts": {
        "start-server": "npm start",
        "test": "mocha e2e-spec.js",
        "ci": "start-server-and-test start-server http://localhost:8080 test"
    }
}
```

もちろん、npx で起動することもでき、この場合は `<SERVER-COMMAND>` と `TEST-COMMAND` の npx は省略可能だ。

```
$ npx start-server-and-test start 8080 test
```

ただし、**サーバに webpack-dev-server を使っている場合は注意が必要**。start-server-and-test はデフォルトで `HEAD` による確認を行うのだが、web pack-dev-server は `HEAD` に対応していない。この問題を回避するために、第 2 引数を **`http-get://` で始まる URL で指定**しよう。

```bash
$ npx start-server-and-test start http-get://localhost:8080/ test
```

他にも [README](https://github.com/bahmutov/start-server-and-test) にはタイムアウトの設定や複数サーバを起動する方法について記載されているので、困ったら参照してほしい。

[^1]: [Google Testing Blog: Flaky Tests at Google and How We Mitigate Them](https://testing.googleblog.com/2016/05/flaky-tests-at-google-and-how-we.html)
[^2]: [Feature request: Add the ability to configure the server wait behavior · Issue #8870 · cypress-io/cypress](https://github.com/cypress-io/cypress/issues/8870)

