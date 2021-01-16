---
title: "Expo の TypeScript プロジェクトで自動テスト"
emoji: "🐄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [expo, reactnative, typescript, jest]
published: true
---

最近は [Expo](https://expo.io/) で [React Native](https://reactnative.dev/) もやってます🙂

今回は、Expo のプロジェクトを Jest でテストしたい。基本的に [Testing with Jest - Expo Documentation](https://docs.expo.io/guides/testing-with-jest/) にしたがって進めればいいのだが、TypeScript の場合は多少設定を変える必要がある。この記事では以下のポイントを紹介する。

1. [Expo の TypeScript プロジェクトで Jest を使うときの設定](#%E3%83%86%E3%82%B9%E3%83%88%E3%81%AE%E5%AE%9F%E8%A1%8C%E3%81%A8-jest-%E3%81%AE%E8%A8%AD%E5%AE%9A%E5%A4%89%E6%9B%B4)
2. [Jest の設定ファイルも TypeScript で書く方法](#jest-%E3%81%AE%E8%A8%AD%E5%AE%9A%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%82%82-typescript-%E3%81%A7%E6%9B%B8%E3%81%8F)

なお、手元の環境は次の通り。

```bash
$ expo --version                                              
4.0.17
$ npm list expo
my-expo-app
└── expo@40.0.0
```

## Jest の設定とテストの用意

まずは、必要なパッケージをインストールする。jest パッケージではなく [jest-expo](https://www.npmjs.com/package/jest-expo) をインストールするのが Expo 流。

```bash
$ npm i jest-expo react-test-renderer --save-dev
```

TypeScript プロジェクトなので、インストールしたパッケージの `.d.ts` もインストールしておこう。

```bash
$ npm i @types/jest @types/react-test-renderer --save-dev
```

Jest の設定は、とりあえずドキュメント通り `package.json` に追加する。

```json
"jest": {
  "preset": "jest-expo",
  "transformIgnorePatterns": [
    "node_modules/(?!(jest-)?react-native|react-clone-referenced-element|@react-native-community|expo(nent)?|@expo(nent)?/.*|react-navigation|@react-navigation/.*|@unimodules/.*|unimodules|sentry-expo|native-base|@sentry/.*)"
  ]
}
```

`transformIgnorePatterns` にやたらと長い正規表現が書かれている🤔

Jest はデフォルトで `node_modules/` 以下をトランスパイル対象から外すのだが、React Native のライブラリはトランスパイルされずに配布されているものがある。[`transformIgnorePatterns`](https://jestjs.io/docs/en/configuration#transformignorepatterns-arraystring) を上記のように設定することで、それらをトランスパイル対象にしている。

::: message

`x(?!y)` 否定先読み。`x` に `y` が続かない場合にマッチする。こういう長い正規表現は Perl のようにコメントをつけて書きたいが、残念ながら `/x` 修飾子は使えない（参考: [正規表現 - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Regular_Expressions)）

:::

では、サンプルのテスト `App.test.tsx` を書いてみよう。

```typescript:App.test.tsx
import React from "react";
import renderer from "react-test-renderer";

import App from "./App";

describe("<App />", () => {
  it("has 1 child", () => {
    const tree = renderer.create(<App />).toJSON();
    // @ts-ignore
    expect(tree.children.length).toBe(1);
  });
});
```

実は、上記のコードでは型エラーが出るのだが、ここでは一旦 `@ts-ignore` している。型エラーの修正は本筋から外れるのと、将来的には react-test-renderer を直接使わずに [React Testing Library](https://testing-library.com/docs/react-testing-library/intro/) を使うつもりだからだ。

## テストの実行と Jest の設定変更

いよいよテストを実行してみよう。

```bash
$ npx jest
 FAIL  ./App.test.tsx
  <App />
    ✕ has 1 child (45ms)

  ● <App /> › has 1 child

    Element type is invalid: expected a string (for built-in components) or a class/function (for composite components) but got: object.

      at createFiberFromTypeAndProps (node_modules/react-test-renderer/cjs/react-test-renderer.development.js:16180:21)
      at createFiberFromElement (node_modules/react-test-renderer/cjs/react-test-renderer.development.js:16208:15)
      at reconcileSingleElement (node_modules/react-test-renderer/cjs/react-test-renderer.development.js:5358:23)
      at reconcileChildFibers (node_modules/react-test-renderer/cjs/react-test-renderer.development.js:5418:35)
      at reconcileChildren (node_modules/react-test-renderer/cjs/react-test-renderer.development.js:7991:28)
      at updateHostRoot (node_modules/react-test-renderer/cjs/react-test-renderer.development.js:8547:5)
      at beginWork (node_modules/react-test-renderer/cjs/react-test-renderer.development.js:9994:14)
      at performUnitOfWork (node_modules/react-test-renderer/cjs/react-test-renderer.development.js:13800:12)
      at workLoopSync (node_modules/react-test-renderer/cjs/react-test-renderer.development.js:13728:5)
      at renderRootSync (node_modules/react-test-renderer/cjs/react-test-renderer.development.js:13691:7)

  console.error
    Warning: React.createElement: type is invalid -- expected a string (for built-in components) or a class/function (for composite components) but got: object.

       6 | describe("<App />", () => {
       7 |   it("has 1 child", () => {
    >  8 |     const tree = renderer.create(<App />).toJSON();
         |                                  ^
       9 |     expect(tree.children.length).toBe(1);
      10 |   });
      11 | });

      at printWarning (node_modules/react/cjs/react.development.js:315:30)
      at error (node_modules/react/cjs/react.development.js:287:5)
      at Object.createElementWithValidation [as createElement] (node_modules/react/cjs/react.development.js:1788:7)
      at Object.<anonymous> (App.test.tsx:8:34)

Test Suites: 1 failed, 1 total
Tests:       1 failed, 1 total
Snapshots:   0 total
Time:        1.254s
Ran all test suites.
```

失敗してしまった。エラーも謎な感じだ。

```
Warning: React.createElement: type is invalid -- expected a string (for built-in components) or a class/function (for composite components) but got: object.
```

`App` コンポーネントが `object` だという警告が出ている。このエラーだが、Jest を実行したときに、

```typescript
import App from "./App";
```

この `import` で **`App.tsx` ではなく `app.json` が読み込まれる**ために起こっている。これを防ぐためには、モジュールの読み込みで `.json` よりも `.tsx`, `.ts` を優先するように Jest の [`moduleFileExtensions`](https://jestjs.io/docs/ja/configuration#modulefileextensions-arraystring) を設定してやる。

```json
"jest": {
  ...
  "moduleFileExtensions": [
    "ts",
    "tsx",
    "js",
    "jsx"
  ]
},
```

この変更を施したあと、再実行すれば無事にテストが通る。

```bash
$ npx jest
 PASS  ./App.test.tsx
  <App />
    ✓ has 1 child (2341ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        4.855s
```

検索してみたところ、他に同様の問題に当たっている人がいるようなので、[^1] [ドキュメントの修正 PR](https://github.com/expo/expo/pull/11580) を投げておいた。

## Jest の設定ファイルも TypeScript で書く

さて、テストを動かすことはできたわけだが、Jest の設定が多少複雑になってしまった。

```json
"jest": {
  "preset": "jest-expo",
  "transformIgnorePatterns": [
    "node_modules/(?!(jest-)?react-native|react-clone-referenced-element|@react-native-community|expo(nent)?|@expo(nent)?/.*|react-navigation|@react-navigation/.*|@unimodules/.*|unimodules|sentry-expo|native-base|@sentry/.*)"
  ],
  "moduleFileExtensions": [
    "ts",
    "tsx",
    "js",
    "jsx"
  ]
}
```

特に `transformIgnorePatterns` は複雑だし、`package.json` に記述するよりは外に出したいところだ。そして、できればTypeScript で書きたい。

Jest [26.6.0](https://github.com/facebook/jest/blob/master/CHANGELOG.md#2660) からは **TypeScript による設定ファイルもサポートされている**。[ts-node](https://www.npmjs.com/package/ts-node) が必要なのでインストールする。

```bash
$ npm i ts-node --save-dev
```

また、jest-expo が依存している jest が古い場合は、こちらもインストールする。

```bash
$ npm list jest           
my-expo-app
└─┬ jest-expo@40.0.1
  └── jest@25.5.4 
$ npm i jest --save-dev 
```

では、新しく `jest.config.ts` を作成し、設定を移していこう。

```typescript:jest.config.ts
import { Config } from "@jest/types";

// By default, all files inside `node_modules` are not transformed. But some 3rd party
// modules are published as untranspiled, Jest will not understand the code in these modules.
// To overcome this, exclude these modules in the ignore pattern.
const untranspiledModulePatterns = [
  "(jest-)?react-native",
  "react-clone-referenced-element",
  "@react-native-community",
  "expo(nent)?",
  "@expo(nent)?/.*",
  "react-navigation",
  "@react-navigation/.*",
  "@unimodules/.*",
  "unimodules",
  "sentry-expo",
  "native-base",
  "@sentry/.*",
];

const config: Config.InitialOptions = {
  preset: "jest-expo",
  transformIgnorePatterns: [
    `node_modules/(?!${untranspiledModulePatterns.join("|")})`,
  ],
  moduleFileExtensions: ["ts", "tsx", "js", "jsx"],
};

export default config;
```

長ったらしい正規表現を分解したことで、かなり見通しがよくなった。コメントが書けない `package.json` に比べると、コメントが書けるだけでも嬉しい😅

最後に一点。jest を jest-expo が依存しているバージョンではなく、改めてインストールしなおした場合、`node_modules` にはふたつのバージョンの jest が混在していることになる。

`npm list` コマンドで確認してみよう。

```bash
$ npm list jest        
my-expo-app
├── jest@26.6.3 
└─┬ jest-expo@40.0.1
  └── jest@25.5.4
```

ふたつのバージョンの Jest が混在していることが分かるだろう。これにはひとつ問題があって、**npx コマンドで実行されるバージョンがどちらになるか分からない**のである。

npx コマンドは `node_modules/.bin` 以下に配置されたコマンドを実行するのだが、これらは各パッケージ内のファイルへのシンボリックリンクになっており、インストール順により上書きされてしまう。

たとえば、今、`./node_modules/.bin/jest` が `./node_modules/jest/bin/jest.js` を指していたとしても、

```bash
$ ls -l ./node_modules/.bin/jest
lrwxr-xr-x  1 takanori_is  staff  19  1 16 16:31 ./node_modules/.bin/jest -> ../jest/bin/jest.js
```

`jest-expo` をインストールしなおせば `./node_modules/jest-expo/bin/jest.js` に置き換わってしまう。

```bash
$ ls -l ./node_modules/.bin/jest
lrwxr-xr-x  1 takanori_is  staff  24  1 16 16:34 ./node_modules/.bin/jest -> ../jest-expo/bin/jest.js
```

なので、`jest-expo` が依存する `jest` のバージョンが上がるまでは、npx コマンド経由ではなく `./node_modules/jest/bin/jest.js` を直接実行するのが安全だろう。`package.json` に script として登録しておこう。

```json
"scripts": {
  ...
  "test": "./node_modules/jest/bin/jest.js"
}
```

`npm test` で実行できるようになった。

```bash
$ npm test                  

> @ test /Users/ishikawasonkyou/Developer/Workspace/my-expo-app
> ./node_modules/jest/bin/jest.js

 PASS  ./App.test.tsx
  <App />
    ✓ has 1 child (150 ms)

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   0 total
Time:        0.767 s, estimated 1 s
Ran all test suites.
```



[^1]: [reactjs - Jest-Expo crashes on example (React.createElement: type is invalid -- expected a string) - Stack Overflow](https://stackoverflow.com/questions/65549722/jest-expo-crashes-on-example-react-createelement-type-is-invalid-expected-a)

