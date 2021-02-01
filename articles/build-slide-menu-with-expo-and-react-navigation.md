---
title: "Expo と React Navigation で Twitter のようなスライドメニューを作る"
emoji: "⛺️"
type: "tech"
topics: [reactnative, expo]
published: false
---

Twitter の iOS アプリは左サイドメニューがスライドによって開閉できるようになっている。

![TwitterSlideMenu](https://raw.githubusercontent.com/ishikawa/my-zenn-content/main/articles/build-slide-menu-with-expo-and-react-navigation/TwitterSlideMenu.png)

普段使い慣れていることもあり、これを　[Expo](https://expo.io/) と [React Native](https://reactnative.dev/) で実装できるか試してみたい。

## React Navigation

Expo でナビゲーションやルーティングを実装するときは [React Navigation](https://reactnavigation.org/) を使うことが[推奨されている](https://docs.expo.io/guides/routing-and-navigation/)。

ただ、[公式のガイド](https://reactnavigation.org/docs/drawer-based-navigation)は非常に基本的な内容に留まっており、これだけだと期待しているようなメニューの実装ができない。実は、ここに書かれている以外にも多くの [API が用意されており](https://reactnavigation.org/docs/drawer-navigator)、これらを組み合わせれば、以下のようなカスタマイズが可能だ。

- メニュを開くとき一緒にメインのビューもスライドさせる
- メニューの幅を調整
- メニューの内容を独自のビューに置き換える

最終的には次のようなアプリケーションが作れるようになる予定である。

*（最終バージョンのスクリーンショット）*

## 新規プロジェクトの準備

新規に Expo プロジェクトを作成する。テンプレートは Managed workflow の "blank" を選択する。

```bash
$ expo init
✔ What would you like to name your app? … my-slide-menu
✔ Choose a template: › blank (TypeScript)    same as blank but with
TypeScript configuration
✔ Downloaded and extracted project files.
🧶 Using Yarn to install packages. Pass --npm to use npm instead.
⠙ Installing JavaScript dependencies.
...
```

作られたプロジェクトのディレクトリに移動して、起動できることを確認しておこう。

```bash
$ expo start
```

## 必要なパッケージのインストール

次に、必要なパッケージのインストールをしていく。まずは React Navigation をインストールする。

```bash
$ expo install @react-navigation/native
```

今回は Expo の Managed workflow で開発しているので、さらに追加のパッケージも必要だ。

```bash
$ expo install react-native-gesture-handler react-native-reanimated react-native-screens react-native-safe-area-context @react-native-community/masked-view
```

また、サイドメニュー（ドロワー）は @react-navigation/drawer で提供されているので、最後にこれを追加しよう。

```bash
$ expo install @react-navigation/drawer
```

## 基本的なメニューを作る

次に、公式ガイドに従って、基本的なサイドメニューを実装する。ドキュメントのサンプルコードに TypeScript の型アノテーションを追加したコードが以下になる。[^1]

```typescript:App.tsx
import React from "react";
import { StyleSheet, Button, Text, View } from "react-native";
import {
  createDrawerNavigator,
  DrawerNavigationProp,
} from "@react-navigation/drawer";
import { NavigationContainer } from "@react-navigation/native";
import { StatusBar } from "expo-status-bar";

type HomeNavigationParamList = {
  Home: undefined;
  Notifications: undefined;
};

type HomeScreenNavigationProp = DrawerNavigationProp<
  HomeNavigationParamList,
  "Home"
>;

type NotificationsScreenNavigationProp = DrawerNavigationProp<
  HomeNavigationParamList,
  "Notifications"
>;

function HomeScreen({ navigation }: { navigation: HomeScreenNavigationProp }) {
  return (
    <View style={{ flex: 1, alignItems: "center", justifyContent: "center" }}>
      <Button
        onPress={() => navigation.navigate("Notifications")}
        title="Go to notifications"
      />
    </View>
  );
}

function NotificationsScreen({
  navigation,
}: {
  navigation: NotificationsScreenNavigationProp;
}) {
  return (
    <View style={styles.container}>
      <Button onPress={() => navigation.goBack()} title="Go back home" />
    </View>
  );
}

const Drawer = createDrawerNavigator<HomeNavigationParamList>();

export default function App() {
  return (
    <NavigationContainer>
      <Drawer.Navigator initialRouteName="Home">
        <Drawer.Screen name="Home" component={HomeScreen} />
        <Drawer.Screen name="Notifications" component={NotificationsScreen} />
      </Drawer.Navigator>
      <StatusBar style="auto" />
    </NavigationContainer>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: "#fff",
    alignItems: "center",
    justifyContent: "center",
  },
});
```

とりあえず動くメニューを実装できた。起動してみよう。

![InitialVersion](https://raw.githubusercontent.com/ishikawa/my-zenn-content/main/articles/build-slide-menu-with-expo-and-react-navigation/InitialVersion.gif)

しかし、このままでは思い描いた動作とは程遠い。

1. **メニューがメインのビューに重なっている**（一緒に移動するようにしたい）
2. **メニューの幅が狭すぎる**（もっと広くしたい）
3. **メニューの内容が固定**（登録済みの Routes が表示されるだけ）

順番に解決していこう。

[^1]: React Navigation の TypeScript による型づけについては [Type checking with TypeScript | React Navigation](https://reactnavigation.org/docs/typescript) を参考