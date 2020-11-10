---
title: "Cypress をサーバ起動まで待機させる Returns"
emoji: "🚀"
type: "tech"
topics: ["javascript", "e2e", "test"]
published: true
---

前回、サーバが起動するまで cypress を待機させるために [start-server-and-test を導入](https://zenn.dev/takanori_is/articles/cypress-wait-for-server-startup)したが、それでも偶に（時と場合によっては頻繁に）エラーが出てしまう。

**環境**

- CircleCI 2.0
- cypress 4.12.1
- start-server-and-test 1.11.5

エラーの出力は以下の通り

```
Error: This socket has been ended by the other party
    at Socket.writeAfterFIN [as write] (net.js:455:14)
    at PoolWorker.writeJson (/home/circleci/front_workspace/node_modules/thread-loader/dist/WorkerPool.js:89:22)
    at PoolWorker.run (/home/circleci/front_workspace/node_modules/thread-loader/dist/WorkerPool.js:69:12)
    at WorkerPool.distributeJob (/home/circleci/front_workspace/node_modules/thread-loader/dist/WorkerPool.js:326:20)
    at /home/circleci/front_workspace/node_modules/thread-loader/node_modules/async/queue.js:10:5
    at Object.process (/home/circleci/front_workspace/node_modules/thread-loader/node_modules/async/internal/queue.js:175:17)
    at /home/circleci/front_workspace/node_modules/thread-loader/node_modules/async/internal/queue.js:82:19
    at Immediate.<anonymous> (/home/circleci/front_workspace/node_modules/thread-loader/node_modules/async/internal/setImmediate.js:27:16)
    at processImmediate (internal/timers.js:456:21)
Emitted 'error' event on Socket instance at:
    at emitErrorNT (net.js:1340:8)
    at processTicksAndRejections (internal/process/task_queues.js:84:21) {
  code: 'EPIPE'
}
error Command failed with exit code 1.
info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.
Error: server closed unexpectedly
    at ChildProcess.onClose (/home/circleci/front_workspace/node_modules/start-server-and-test/src/index.js:69:14)
    at ChildProcess.emit (events.js:315:20)
    at maybeClose (internal/child_process.js:1021:16)
    at Process.ChildProcess._handle.onexit (internal/child_process.js:286:5)
```

1. start-server-and-test がサブプロセスとして webpack-dev-server を起動
2. プロセスの起動に [execa](https://github.com/sindresorhus/execa) を使っている
3. コンパイル中にサブプロセスの stdio が閉じられて start-server-and-test が[エラーを発生させる](https://github.com/bahmutov/start-server-and-test/blob/0d3d11e26c5f1f14588006841453ac7146ead0ad/src/index.js#L68)

動作確認のために CircleCI のイメージを使ってローカルでも試してみる。

```bash
$ docker run --rm -it -w /code -v $(pwd):/code circleci/node:12.18-browsers /bin/bash
```

しかし、ローカルで実行すると安定して動いてしまう。

## 原因は webpack の worker 数

webpack-dev-server の出力を抑えたり、CircleCI で失敗したステップのリトライを実装したり、curl で再実装したりと試行錯誤を繰り返してみたが、なかなか安定しない。しかし、何度か試すうちに **webpack-dev-server 単体でも EPIPE が発生している**ことを確認できた。

Webpack は全くの素人なのだが、[Webpackのビルド時間を短くするための取り組み - freee Developers Blog](https://developers.freee.co.jp/entry/2017/12/25/163006) によると、ビルドには worker プロセスを起動しているようなので、このへんが怪しいと推測（サブプロセスと通信しており、それなら EPIPE が発生しそう）。

[thread-loader](https://github.com/webpack-contrib/thread-loader) のドキュメントを斜め読みしながら、手元の環境の設定を確認してみると以下のようになっていた。

```javascript
const workerPool = {
  workers: require('os').cpus().length - 1,
  poolTimeout: Infinity,
  workerParallelJobs: 30,
  name: 'build-js-pool'
};
```

これだと、CI 環境で報告される CPU コア数が多いと簡単にメモリ不足になりそうだ。実際に、CircleCI で実行するときに `workerPool` の中身を確認すると `workers = 35` だった。さすがにこれは多すぎる。

最終的に、以下のように変更することで対処した。

```Javascript
const workerPool = {
  workers: (process.env.CI ? 4 : require('os').cpus().length - 1),
  poolTimeout: Infinity,
  workerParallelJobs: 30,
  name: 'build-js-pool'
};
```

もちろん、`4` は適当な数なので各自の環境に合わせて調整してほしい。