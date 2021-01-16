---
title: "Expo で PKCE を使って Auth0 連携"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["reactnative", "auth0", "expo"]
published: false
---

[Expo](https://expo.io/) で [Auth0](https://auth0.com/) を IdP (**Id**entity **P**rovider) としてログインするには、

- WebBrowser.maybeCompleteAuthSession()
- OIDC Discovery
- nonce 生成
- PKCE
- nonce 検証