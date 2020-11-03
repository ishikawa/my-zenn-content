---
title: "@babel/plugin-transform-instanceof を読む"
emoji: "🐡"
type: "tech"
topics: ["javascript", "babel"]
published: true
---

[Babel](https://babeljs.io/) の[プラグイン](https://babeljs.io/docs/en/plugins)開発について勉強しはじめた。

まずは手近な実装として Babel が提供しているプラグインのうち、[@babel/plugin-transform-instanceof](https://babeljs.io/docs/en/babel-plugin-transform-instanceof) を読んでみる。このプラグインは、ES6 の [instanceof 演算子](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/instanceof)を変換するだけなので[実装コード](https://github.com/babel/babel/blob/v7.12.4/packages/babel-plugin-transform-instanceof/src/index.js)も短く読みやすい。

:::message

なお、参照している Babel のバージョンは [7.12.4](https://github.com/babel/babel/blob/v7.12.4/) である

:::

## @babel/helper-plugin-utils

```Javascript
import { declare } from "@babel/helper-plugin-utils";

export default declare(api => {
  api.assertVersion(7);

  ...
});
```

まずは、モジュールから export されている箇所だが、[@babel/helper-plugin-utils](https://babeljs.io/docs/en/babel-helper-plugin-utils.html) から [`declare`](https://github.com/babel/babel/blob/v7.12.4/packages/babel-helper-plugin-utils/src/index.js#L1) 関数が使われている。これは Web サイトにも解説があるとおり、Babel 7 と以前のバージョンの互換性を保つためのものだ。この `declare` 関数に渡された関数は、以下の修正が施されるように別関数にラップされて返される。

- 第 1 引数 `api` ([babel](https://github.com/babel/babel/tree/master/packages/babel-core) オブジェクト) が `assertVersion()` メソッドを持つようになる
- 第 2 引数 `options` が常に渡されるようになる

## Visitor オブジェクト

```javascript
export default declare(api => {
  return {
    name: 'transform-instanceof',
    visitor: {...}
  };
});
```

export した関数からは `visitor` プロパティをもつオブジェクトを返すようにする。`visitor` プロパティに指定したオブジェクトのメソッドが AST のノードに応じて呼び出されるわけだが、このへんは [Babel Plugin Handbook](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/en/plugin-handbook.md#toc-writing-your-first-babel-plugin) に詳しい。

## instanceof 演算子に対応させる

```javascript
visitor: {
  BinaryExpression(path) {
    const { node } = path;
    if (node.operator === 'instanceof') {
      ...
    }
  }
}
```

`instanceof` 演算子を変換したいので、visitor オブジェクトは `BinaryExpression` メソッドを実装し、`path.node.operator` で演算子を判定する。ちなみに、この `BinaryExpression` メソッドは、

```javascript
visitor: {
  BinaryExpression: {
    enter() {...}
    exit() {...}
  }
}
```

のように、[enter/exit を実装することもできる](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/en/plugin-handbook.md#visitors)。

## helper の利用

ここから、実際の変換処理に入る。

```javascript
const helper = this.addHelper('instanceof');
```

export した関数が実行されるときの `this` は [PluginPass](https://github.com/babel/babel/blob/v7.12.4/packages/babel-core/src/transformation/plugin-pass.js) であり、この [`addHelper()`](https://github.com/babel/babel/blob/v7.12.4/packages/babel-core/src/transformation/plugin-pass.js#L39) 関数は以下のように、`file` オブジェクトにそのまま委譲されている。

```javascript:file.js
addHelper(name: string) {
  return this.file.addHelper(name);
}
```

この addHelper() 関数の実装は[割と複雑](https://github.com/babel/babel/blob/v7.12.4/packages/babel-core/src/transformation/file/file.js#L166)なのだが、動作は [@babel/helpers](https://babeljs.io/docs/en/babel-helpers) に説明がある。

> // The .addHelper function adds, if needed, the helper to the file
> // and returns an expression which references the helper

理解できた範囲でまとめると、

- コアのプラグインは @babel/helpers を使って、変換に必要な関数を定義している
- たとえば、instanceof を実装する関数は[こんな感じ](https://github.com/babel/babel/blob/v7.12.4/packages/babel-helpers/src/helpers.js#L602)
- 関数の実装など長めのコードを構築するには [@babel/template](https://babeljs.io/docs/en/babel-template) が便利

addHelper() 関数のメモ

- ローカル変数の binding を管理する [Scope](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/en/plugin-handbook.md#scope)
- path の [unshiftContainer, pushContainer](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/en/plugin-handbook.md#inserting-into-a-container) - body のようなコンテナノードの開始位置、終了位置にノードを追加

## helper の利用 (2)

```javascript
const isUnderHelper = path.findParent(path => {
  return (
    (path.isVariableDeclarator() && path.node.id === helper) ||
    (path.isFunctionDeclaration() && path.node.id && path.node.id.name === helper.name)
  );
});

if (isUnderHelper) {
  return;
}
```

ここでは `isUnderHelper` 変数で「走査中のノードがヘルパー自体ではないか」を判定している。これは、たとえば、以下の変換後のコードを見れば理解できると思う。

```javascript
function _instanceof(left, right) { if (right != null && typeof Symbol !== "undefined" && right[Symbol.hasInstance]) { return !!right[Symbol.hasInstance](left); } else { return left instanceof right; } }

_instanceof(this, String)
```

ここで、`_instanceof` 関数がヘルパーとして追加された関数だが、当然、この関数の中身の `instanceof` 演算子を変換してはいけない。なので、`isUnderHelper` 変数を使って判定する必要があるわけだ。

## ノードの置き換え

```Javascript
path.replaceWith(t.callExpression(helper, [node.left, node.right]));
```

最後に、変換対象のノードを置き換える。

1. [@babel/types](https://babeljs.io/docs/en/babel-types) の [callExpression()](https://babeljs.io/docs/en/babel-types#callexpression) でヘルパー関数を呼び出すためのノードを生成
2. [path.replaceWith()](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/en/plugin-handbook.md#replacing-a-node) によって対象ノードを置き換え

