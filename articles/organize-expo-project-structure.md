---
title: "Expo プロジェクトのソースコードの構成を変える"
emoji: "🐮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [expo, reactnative]
published: false
---

（前巻までのあらすじ）[前回の記事](https://zenn.dev/takanori_is/articles/setup-jest-for-expo-typescript-project)では、初期状態の Expo プロジェクトに Jest によるテストを追加した。TypeScript の場合は公式ガイドの手順に加えて、若干の修正が必要だった。そして、この時点でのファイル構成は以下のようになっている…*!!*

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
├── tsconfig.json
└── yarn.lock

2 directories, 9 files
```

今回はこのファイル構成を以下のように変更したい。

1. ソースコードを `src` ディレクトリと `test` ディレクトリに分ける
2. エントリーポイントを `src/App.tsx` に変更