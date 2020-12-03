---
title: "React Testing Library で awaitable な Query を使いたい"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "test"]
published: true
---

React Testing Library で `findBy*` を使って、要素が表示されるのを待機したいのだが、いざ使ってみると失敗する。

```javascript
await screen.findByTestId('the-element');
```

これを実行すると、以下のような Warning が出る。

```
  console.error node_modules/@testing-library/react/dist/act-compat.js:88
    It looks like you're using a version of react-dom that supports the "act" function, but not an awaitable version of "act" which you will need. Please upgrade to at least react-dom@16.9.0 to remove this warning.
```

この警告は、React 16.8 で追加された[^1] [ReactTestUtils.act()](https://reactjs.org/docs/testing-recipes.html#act) がその時点では async 関数をサポートしておらず、16.9 でサポートされた[^2] ので、react-dom をアップグレードしろ、という内容である。

:::message 

もっとも、慌てて 16.9 にアップグレードしなくても動作に問題はない。React 16.9 にすると Deprecate な ライフサイクル・メソッドで警告が出るようになるので、すぐにアップグレードが難しい場合もあるだろう。

:::

react-dom のバージョンを上げるのだが、react と react-test-renderer のバージョンも合わせる必要がある。今回は 16.8 を使っていたので、16.9.0 にする。

```javascript
"react": "16.9.0",
"react-dom": "16.9.0",
"react-test-renderer": "16.9.0",
```

しかし、これで再度実行すると、今度は以下のエラーが出る。

```
TypeError: MutationObserver is not a constructor
```

Jsdom を 14.x にする必要がある。[^3] しかし、jest が使っているデフォルトの jsdom が古いので、[jest-environment-jsdom-fourteen](https://developer.aliyun.com/mirror/npm/package/jest-environment-jsdom-fourteen) を導入する。まず、このパッケージをインストール。

```
$ npm install --save-dev jest-environment-jsdom-fourteen
```

次に Jest の設定に以下を追加する。

```json
{
  "testEnvironment": "jest-environment-jsdom-fourteen"
}
```



[^1]: [React v16.8: The One With Hooks – React Blog](https://ja.reactjs.org/blog/2019/02/06/react-v16.8.0.html)
[^2]: [React v16.9.0 and the Roadmap Update – React Blog](https://ja.reactjs.org/blog/2019/08/08/react-v16.9.0.html#async-act-for-testing)
[^3]: [TypeError: MutationObserver is not a constructor · Issue #731 · testing-library/react-testing-library](https://github.com/testing-library/react-testing-library/issues/731)



