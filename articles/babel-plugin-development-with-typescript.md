---
title: "Babel プラグインを TypeScript で開発するときの型"
emoji: "⚡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "babel"]
published: true
---

Babel のプラグイン開発は、JavaScript や Flow/TypeScript のノードに対応した多くの API があり、API リファレンスを調べるだけでも面倒。どうせなら、VSCode の強力なサポートが得られる TypeScript で書きたい。

ということで、どんなふうに TypeScript で Babel プラグインを書いているかと、いくつか注意する点を紹介する。

:::message

Babel のバージョンは 7.12

:::

## パッケージ導入

特に複雑なことをしなければ、`@babel/core` だけで十分

```bash
$ npm i @babel/core
```

開発用に TypeScript コンパイラと Babel CLI。あとは必要な型定義を導入

```bash
$ npm i typescript @babel/cli @types/babel__core
```

**`@types/babel-core` というパッケージも存在するので注意。** こちらの定義は古いので、[`@types/babel__core`](https://www.npmjs.com/package/@types/babel__core) を入れましょう。

## コード

最低限、types と NodePath, PluginObj の型があればいい。

```typescript
import { types as t } from '@babel/core';
import type { NodePath, PluginObj } from '@babel/core';
```

Visitor はこんな感じ。`NodePath` に適切な型パラメータを与えれば、`path.node` も期待通りの型になる。

```typescript
const visitor = {
  ClassDeclaration(path: NodePath<t.ClassDeclaration>) {
    const { node } = path;  // const node: t.ClassDeclaration
    // ...
  }
};

```

ノードの種別判定には、`types.isXXX`() を使おう。TypeScript の [type predicates](https://www.typescriptlang.org/docs/handbook/advanced-types.html#using-type-predicates) のおかげで、`types.isXXX()` を通過したあとのコードパスでは node が適切な型として認識される。

```typescript
if (
  t.isCallExpression(node.expression) &&  // node.expression: t.Expression
  t.isSuper(node.expression.callee) &&    // node.expression: t.CallExpression
  node.expression.arguments.length === 1 &&
  t.isIdentifier(node.expression.arguments[0], { name: 'props' })
) {
  // ...
}
```

最後に、このモジュールから返す関数の型定義は多少トリッキーだ。

```typescript
function transform(_api: typeof t): PluginObj {
  return {
    name: 'babel-example-plugin',
    visitor
  };
}

export default transform;
```

`_api` として渡されるのは `@babel/core` の `types` なのだが、これは、

```typescript
import * as t from '@babel/types';
...
export { ParserOptions, GeneratorOptions, t as types, template, traverse, NodePath, Visitor };
```

このように、[namespace imports](https://developer.mozilla.org/en-US/docs/web/javascript/reference/statements/import) されたシンボルがエクスポートされており、そのままでは型としてい使用できない。そのため、ここでは [`typeof` 演算子を使って](https://basarat.gitbook.io/typescript/type-system/moving-types#capturing-the-type-of-a-variable)いる（もっとも、この例では `_api` を使っていないのだが...）

## 参考

- [esamattis/babel-plugin-ts-optchain: Babel plugin for transpiling legacy browser support to ts-optchain by removing Proxy usage.](https://github.com/esamattis/babel-plugin-ts-optchain) - TypeScript で書かれた babel plugin
- [Typesafe babel plugin development with TypeScript · Issue #10637 · babel/babel](https://github.com/babel/babel/issues/10637)

