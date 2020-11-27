---
title: "モノレポで複数パッケージを管理するときの tsconfig"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

モノレポで複数パッケージを管理する場合、tsconfig の `extends` を使って設定を共通化したい。[babel のリポジトリ](https://github.com/babel/babel/tree/main/packages)が参考になる。

まずはトップレベルに、共通設定をまとめた `tsconfig.base.json` を置く。この tsconfig も `@tsconfig` から適切なものを継承することになるだろう。

```json:tsconfig.base.json
{
  "extends": "@tsconfig/node12/tsconfig.json",
  "compilerOptions": {
    "esModuleInterop": true,
    "sourceMap": true,
    "lib": ["es2019", "es2020.promise", "es2020.bigint", "es2020.string", "dom"]
  }
}
```

ここで注意したいのは、ファイルパスの扱いだ。[TSConfig リファレンス](https://www.typescriptlang.org/tsconfig#extends) によると、

> All relative paths found in the configuration file will be resolved relative to the configuration file they originated in.

とあり、例えば、`tsconfig.base.json` に `compilerOptions.outDir` が設定されている、とする。

```json:tsconfig.base.json
{
  "compilerOptions": {
    "outDir": "dist",
    ...
  },
  ...
}
```

この場合、`tsconfig.base.json` と同じディレクトリにある `dist` ディレクトリが設定される。これは期待している結果ではないはずなので、ファイルパスの設定は各パッケージごとの `tsconfig.json` で指定するのがいいだろう。

```json:tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "typeRoots": ["./node_modules/@types"]
  },
  "exclude": ["node_modules", "**/*.spec.ts"]
}
```

