---
title: "Expo と React Navigation で Twitter のスライドメニューを作る"
emoji: "⛺️"
type: "tech"
topics: [reactnative, expo]
published: false
---

[Expo](https://expo.io/) でナビゲーションを実装するときは React Navigation を使うことが[推奨されている](https://docs.expo.io/guides/routing-and-navigation/)。今回は、React Navigation を使って、スライドで開閉できるサイドメニューを実装したい。これはたとえば、iOS の Twitter アプリで見られるようなメニューである。

（写真）

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

また、今回作ろうとしているサイドメニュー（ドロワー）は @react-navigation/drawer で提供されているので、最後にこれを追加しよう。

```bash
$ expo install @react-navigation/drawer
```

## 基本的なメニューを作る

次に、公式ドキュメントの [Drawer navigation | React Navigation](https://reactnavigation.org/docs/drawer-based-navigation) に従って、基本的なサイドメニューを実装する。ドキュメントのサンプルコードに TypeScript の型アノテーションを追加したコードが以下になる。

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

