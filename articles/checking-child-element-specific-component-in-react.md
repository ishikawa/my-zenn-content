---
title: "React で子要素が特定のコンポーネントかどうかを調べる"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

まず、props で渡された `children` を [React.Children.toArray](https://ja.reactjs.org/docs/react-api.html#reactchildrentoarray) で配列に変換する。

```js
const childElements = React.Children.toArray(children);
```

`null` は正当な React.Node なので、ここで null チェックは必要ない。

次に、ReactElement の `type` プロパティをチェックすることでコンポーネントのクラス/関数を確認できるのだが、`toArray()` で返ってくる配列の要素は ReactElement とは限らないので [React.isValidElement()](https://ja.reactjs.org/docs/react-api.html#isvalidelement) を使って ReactElement かどうかを確認する。

```js
const lastElement = childElements[childElements.length - 1];

if (React.isValidElement(lastElement)) {
  if (lastElement.type === MyComponent) {
    console.log('The last element is MyComponent');
  }
}
```

配列への範囲外アクセスは単に `undefined` を返すので境界チェックは必要ない。また、TypeScript では、`isValidElement()` が型ガードとして実装されているので、`lastElement.type` にアクセスしてもコンパイルエラーにはならない。

```typescript
function isValidElement<P>(object: {} | null | undefined): object is ReactElement<P>;
```

