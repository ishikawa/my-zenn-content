---
title: "TypeScript の型定義ファイルにないメソッドを追加する"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

TypeScript でプログラムを書くときに重要なのは…そう、型ですよね？

自分が書いているプログラムは当然 TypeScript で書かれているので、型が定義されている。一方、ライブラリも TypeScript で書かれているのであればいいのだが、もちろん、世の中には型のない JavaScript で書かれたライブラリが多く存在する。その場合、TypeScript の型は `.d.ts` というファイルで定義されている[^1]。

たとえば、以下のようなモジュールの実装は、

```javascript
const maxInterval = 12;

function getArrayLength(arr) {
  return arr.length;
}

module.exports = {
  getArrayLength,
  maxInterval,
};
```

次のような `.d.ts` で型を定義できる。

```typescript
export function getArrayLength(arr: any[]): number;
export const maxInterval: 12;
```

既存の JavaScript ライブラリには、こうした `.d.ts` ファイルが [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped) というリポジトリか、ライブラリ自身で提供されているのが現状だ。

## 型定義ファイルにないメソッド

ところで、DefinitelyTyped で提供されている `.d.ts` にはライブラリにあるメソッドが定義されていないことがある。今回、具体的には、[@babel/traverse](https://babeljs.io/docs/en/babel-traverse) の [`Scope#crawl()`](https://github.com/babel/babel/blob/main/packages/babel-traverse/src/scope/index.js#L795) メソッドが見当たらなかった。

こういうときにどうするか、というと、もちろん DefinitelyTyped にプルリクエストを送るのが一番いいのだが、マージされるまでにはある程度の時間がかかる。こういう場合は、もちろん、`@ts-ignore` コメントでコンパイルエラーを無視することもできるのだが、

```typescript
// @ts-ignore
path.scope.crawl();
```

それよりも、[Module Augumentation](https://www.typescriptlang.org/docs/handbook/declaration-merging.html#module-augmentation) で**既存のモジュールの型定義に型定義を追加することができる**。

これを使って、`Scope#crawl()` を追加してみる。

```typescript
declare module "babel__traverse" {
  interface Scope {
    crawl(): void;
  }
}
```

一時凌ぎではあるが、これでコンパイラのチェックも効くし、そのあいだに [DefinitelyTyped にプルリクエストを送ることができる](https://github.com/DefinitelyTyped/DefinitelyTyped/pull/49715)。

[^1]: [TypeScript: Documentation - Introduction](https://www.typescriptlang.org/docs/handbook/declaration-files/introduction.html)

