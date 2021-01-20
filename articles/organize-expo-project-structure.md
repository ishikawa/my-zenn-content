---
title: "Expo プロジェクトでソースコードの構成を変える"
emoji: "🐮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [expo, reactnative]
published: false
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

のためだと説明されているが、幸い、Expo プロジェクトではある程度の下地ができていることもあり、`src` と `test` を分けても、それほど複雑な設定にはならない。

## `src` ディレクトリと `test` ディレクトリに分ける

では早速、`src` ディレクトリと `test` ディレクトリを作成して、`App.tsx` と `App.test.tsx` を移動しよう。

```bash
$ mkdir src test
$ mv App.tsx ./src
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
│   └── App.tsx
├── test
│   └── App.test.tsx
└── tsconfig.json

747 directories, 12 files
```

当然、ファイルパスを変えたので、*Relative import* が動かなくなってる。

```typescript
$ npx tsc
test/App.test.tsx:4:17 - error TS2307: Cannot find module './App' or its corresponding type declarations.

4 import App from "./App";
                  ~~~~~~~
```

`import App from "../src/App";` として修正することもできるが、どうせなら tsconfig.json の `baseUrl` を設定して、*Non-relative import* でインポートできるようにしておこう。

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

TBD

## Expo の設定を変更

TBD