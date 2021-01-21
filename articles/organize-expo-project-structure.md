---
title: "Expo プロジェクトでソースコードの構成を変える"
emoji: "🐮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [expo, reactnative]
published: true
---

*これまでのあらすじ…*

[前回の記事](https://zenn.dev/takanori_is/articles/setup-jest-for-expo-typescript-project)では、初期状態の Expo プロジェクトに Jest によるテストを追加した。TypeScript の場合は公式ガイドの手順に加えて、若干の修正が必要だった。そして、この時点でのファイル構成は以下のようになっている。

```
$ tree -L 1 .
.
├── App.test.tsx
├── App.tsx
├── app.json
├── assets
├── babel.config.js
├── jest.config.ts
├── node_modules
├── package-lock.json
├── package.json
└── tsconfig.json

2 directories, 8 files
```

今回はこのファイル構成を以下のように変更したいと思う。

1. ソースコードを `src` ディレクトリと `test` ディレクトリに分ける
2. エントリーポイントを `src/App.tsx` に変更

なお、`src` と `test` を分けるこの構成は個人的に慣れてるから採用しただけで、たとえば、[Jest - TypeScript Deep Dive](https://basarat.gitbook.io/typescript/intro-1/jest) では `src` 以下にすべての TypeScript ファイルを配置することが推奨されている。TypeScript Deep Dive では、この構成は、

> for a clean project setup.

のためだと説明されているが、幸い、Expo プロジェクトではある程度の下地ができていることもあり、`src` と `test` を分けても複雑な設定にはならない。

## `src` ディレクトリと `test` ディレクトリに分ける

では早速、`src` ディレクトリと `test` ディレクトリを作成して、ファイルを移動しよう。

```bash
$ mkdir src test
$ mv assets App.tsx ./src
$ mv App.test.tsx ./test
```

この時点でのファイル構成は次の通り。

```bash
$ tree -L 2 .
.
├── app.json
...
├── package-lock.json
├── package.json
├── src
│   ├── App.tsx
│   └── assets
├── test
│   └── App.test.tsx
└── tsconfig.json
```

当然、ファイルパスを変えたので、*Relative import* が動かなくなってる。

```typescript
$ npx tsc
test/App.test.tsx:4:17 - error TS2307: Cannot find module './App' or its corresponding type declarations.

4 import App from "./App";
                  ~~~~~~~
```

`import App from "../src/App";` として修正することもできるが、どうせなら tsconfig.json の `baseUrl` を設定して、*Non-relative import* でインポートできるようにしておこう。[^1]

```json:tsconfig.json
{
  "compilerOptions": {
    ...
    "baseUrl": "./src"
  }
}
```

これで、App.test.tsx 側の `import` を以下のようにできる。

```typescript:App.test.tsx
import App from "App";
```

tsc でもエラーは見つからない。

```bash
$ npx tsc
```

## テストを通す

さて、TypeScript の型検査は通ったので、次はテストを実行してみよう。

```typescript
$ npm test

FAIL  test/App.test.tsx
  ● Test suite failed to run

    Cannot find module 'App' from 'test/App.test.tsx'

    However, Jest was able to find:
    	'./App.test.tsx'

    You might want to include a file extension in your import, or update your 'moduleFileExtensions', which is currently ['ts', 'tsx', 'js', 'jsx'].

    See https://jestjs.io/docs/en/configuration#modulefileextensions-arraystring

      2 | import renderer from "react-test-renderer";
      3 | 
    > 4 | import App from "App";
        | ^
```

`App` モジュールが見つからない、と怒られてしまった。TypeScript のコンパイラは通っているのに何故だろう？

実は、Expo (React Native) のプロジェクトでは、TypeScript の変換には TypeScript コンパイラ (`tsc`) ではなく、[Babel を使っている](https://reactnative.dev/docs/javascript-environment#javascript-syntax-transformers)。そして、Babel の TypeScript 変換プラグインである [@babel/plugin-transform-typescript](https://babeljs.io/docs/en/babel-plugin-transform-typescript) は tsconfig.json を読まない。[^2]

そのため、ここでは [babel-plugin-module-resolver](https://github.com/tleunen/babel-plugin-module-resolver) を使って、ルートを `src` ディレクトリに設定しよう。[^3]

```js:babel.config.js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: ["babel-preset-expo"],
    plugins: [
      [
        "module-resolver",
        {
          root: ["./src/"],
          extensions: [".ts", ".tsx", ".mjs", ".js", ".jsx"],
        },
      ],
    ],
  };
};
```

これでテストが通るようになった。

```bash
$ npm test

PASS  test/App.test.tsx
  <App />
    ✓ has 1 child (2568 ms)
```

## Expo の設定を変更

では、`expo start` で Expo の開発サーバを起動して、アプリを開いてみよう。

```bash
Failed to compile.
ENOENT: no such file or directory, open '/Users/ishikawasonkyou/Developer/Workspace/my-zenn-content/.work/my-expo-app/assets/favicon.png'
Unable to resolve asset "./assets/icon.png" from "icon" in your app.json or app.config.js
(node:73125) [DEP0066] DeprecationWarning: OutgoingMessage.prototype._headers is deprecated
Error: Problem validating asset fields in app.json. See https://docs.expo.io/
 • Field: splash.image - cannot access file at './assets/splash.png'.
 • Field: android.adaptiveIcon.foregroundImage - cannot access file at './assets/adaptive-icon.png'.
 • Field: icon - cannot access file at './assets/icon.png'.
Failed building JavaScript bundle.
Unable to resolve "../../App" from "node_modules/expo/AppEntry.js"
```

assets と App モジュールが見つからない、というエラーが出る。まずは、app.json で assets へのパスを修正しておく。

```json:app.json
{
  "expo": {
    ...
    "icon": "./src/assets/icon.png",
    "splash": {
      "image": "./src/assets/splash.png",
      ...
    },
    ...
    "android": {
      "adaptiveIcon": {
        "foregroundImage": "./src/assets/adaptive-icon.png",
        ...
      }
    },
    "web": {
      "favicon": "./src/assets/favicon.png"
    }
  }
}
```

次は、App モジュールをエントリーポイントとして登録する必要がある。これは package.json での `"main"` フィールドの変更と、App モジュール側で `registerRootComponent()` を呼ぶ必要がある。[^4]

```json:package.json
{
  "main": "src/App.tsx",
  ...
}
```

```typescript:App.tsx
import { registerRootComponent } from "expo";
...
export default function App() {
  ...
}
...
registerRootComponent(App);
```

これで問題なくアプリが開くはず。

[^1]: Relative と Non-relative imports について [TypeScript: Documentation - Module Resolution](https://www.typescriptlang.org/docs/handbook/module-resolution.html#relative-vs-non-relative-module-imports)
[^2]: Caveats の節を参照 [@babel/plugin-transform-typescript · Babel](https://babeljs.io/docs/en/babel-plugin-transform-typescript#caveats)
[^3]: このプラグインは [babel-preset-expo](https://github.com/expo/expo/tree/master/packages/babel-preset-expo) に含まれている。また、React Native のガイドでも同様の方法が採られていた [Using TypeScript with React Native · React Native](https://reactnative.dev/docs/0.61/typescript#using-custom-path-aliases-with-typescript)
[^4]: [registerRootComponent - Expo Documentation](https://docs.expo.io/versions/latest/sdk/register-root-component/)