---
title: "Expo プロジェクトのソースコードの構成を変える"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [expo, reactnative]
published: false
---

Expo のプロジェクトは初期状態だと次のような構成になっている。

```
$ tree -L 1 .     
.
├── App.tsx
├── app.json
├── assets
├── babel.config.js
├── node_modules
├── package.json
├── tsconfig.json
└── yarn.lock

2 directories, 6 files
```

今回はこの構成を以下のように変更したい。

1. 
2. エントリーポイントを `src/App.tsx` に