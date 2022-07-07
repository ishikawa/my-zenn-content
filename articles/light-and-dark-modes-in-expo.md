---
title: "Expo プロジェクトでダークモードに対応する"
emoji: "🌗"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["expo", "reactnative"]
published: true
---

Expo のプロジェクトでダークモード対応したときのメモ。利用している環境などは以下の通り。

- Expo SDK 45
- EAS Build [^1] と Custom Development Client [^2] を使用 (Manged Workflow)
- 設定ファイルは `app.config.js` (デフォルトは `app.json`)

基本的には、公式ドキュメントの [Light and Dark modes - Expo Documentation](https://docs.expo.dev/guides/color-schemes/) に従って進めていく。

## 設定とライブラリ追加

デフォルトの状態ではライトモードのみの対応となっているため、OS 側の設定をダークモードにしてもアプリ側では対応できない。今回はライトモード/ダークモード両方に対応できるように設定してみる。

`app.config.js` の `expo.userInterfaceStyle` に `automatic` を指定。

```typescript
export default {
  expo: {
    userInterfaceStyle: 'automatic',
    ...
```

今回のプロジェクトでは、EAS Build で Custom Development Client をビルドしているため、`expo-system-ui` の追加も必要。

```
$ expo install expo-system-ui
```

**plugin の指定も忘れずに**。`expo-system-ui` は Config plugin で app.config.js の `userInterfaceStyle` を読み込んでいる。[^3]

```typescript
export default {
  expo: {
    plugins: [
      'expo-system-ui',
    ],
    ...
```



新しいネイティブライブラリを追加したので、Custom Development Client を再ビルド。余談だが、Custom Development Client の再ビルド時間がまあまあ長い（有料プランでも 10 分前後）。

## ダークモード設定を取得

[Appearance](https://reactnative.dev/docs/appearance) モジュール、または [useColorScheme](https://reactnative.dev/docs/usecolorscheme) フックで、現在のカラースキームを取得できる。

```typescript
import { useColorScheme } from 'react-native';

const colorScheme = useColorScheme();

if (colorScheme === 'dark') {
  // dark mode
}
```

また、Appearance モジュールの [addChangeListener()](https://reactnative.dev/docs/appearance#addchangelistener) を使えば、カラーモードの変更を検知できる。

## NativeBase でのダークモード対応

現在関わっているプロジェクトでは、UI フレームワークとして [NativeBase](https://nativebase.io/) を使っているので、NativeBase でダークモードに対応する方法も調べておく。

NativeBase でもデフォルトはライトモードになっているので、OS 側の設定に追従させるには、`extendTheme` で `useSystemColorMode` を `true` にする必要がある。[^4]

```typescript
import { NativeBaseProvider, extendTheme } from 'native-base';

const customTheme = extendTheme({
  config : {
    useSystemColorMode: true,
  }
});

export default function App() {
  return (
    <NativeBaseProvider theme={customTheme}>
      ...
```

また、前段で説明した Expo 側の設定も必要なことに注意。NativeBase は内部的に `useColorScheme()` を使っている。[^5]

コンポーネントでダークモードに対応するためには、`_dark` 擬似 props を使って「ダークモード時の色」を指定する。

```react
<Heading
  mt="1"
  color="coolGray.600"
  _dark={{
    color: 'warmGray.200',
  }}
  fontWeight="medium"
  size="xs">Welcome</Heading>
```



---



[^1]: [EAS Build - Expo Documentation](https://docs.expo.dev/build/introduction/)
[^2]: [Introducing: Custom Development Clients | by TC Davis | Exposition](https://blog.expo.dev/introducing-custom-development-clients-5a2c79a9ddf8)
[^3]: https://github.com/expo/expo/blob/main/packages/expo-system-ui/plugin/src/withIosUserInterfaceStyle.ts
[^4]: [Color mode (Dark Mode) | NativeBase](https://docs.nativebase.io/color-mode)
[^5]: https://github.com/GeekyAnts/NativeBase/blob/2af374e586034366dcefce9a0f23983836a7901f/src/core/color-mode/hooks.tsx#L30
