---
title: "Expo と React Navigation で Twitter のようなスライドメニューを作る"
emoji: "⛺️"
type: "tech"
topics: [reactnative, expo]
published: true
---

Twitter の iOS アプリは左サイドメニューがスライドによって開閉できるようになっている。

![TwitterSlideMenu](https://raw.githubusercontent.com/ishikawa/my-zenn-content/main/articles/build-slide-menu-with-expo-and-react-navigation/TwitterSlideMenu.png)

普段使い慣れていることもあり、これを　[Expo](https://expo.io/) と [React Native](https://reactnative.dev/) で実装できるか試してみたい。

## React Navigation

Expo でナビゲーションやルーティングを実装するときは [React Navigation](https://reactnavigation.org/) を使うことが[推奨されている](https://docs.expo.io/guides/routing-and-navigation/)。

::: message

以降では、用語を公式ドキュメントと合わせるために、サイドメニューを「**ドロワー**」、メインの画面を「**スクリーン**」と書きます。

:::

ただ、[公式のガイド](https://reactnavigation.org/docs/drawer-based-navigation)は非常に基本的な内容に留まっており、これだけだと期待しているようなメニューの実装ができない。実は、ここに書かれている以外にも多くの [API が用意されており](https://reactnavigation.org/docs/drawer-navigator)、これらを組み合わせれば、以下のようなカスタマイズが可能だ。

- ドロワーを開くとき一緒にスクリーンもスライドさせる
- ドロワーの幅を調整
- ドロワーの内容を独自のビューに置き換える

最終的には次のようなアプリケーションが作れるようになる予定である。

![CustomDrawer](https://raw.githubusercontent.com/ishikawa/my-zenn-content/main/articles/build-slide-menu-with-expo-and-react-navigation/CustomDrawer2.gif)

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

また、ドロワーは `@react-navigation/drawer` で提供されているので、最後にこれを追加しよう。

```bash
$ expo install @react-navigation/drawer
```

## 基本的なメニューを作る

次に、公式ガイドに従って、基本的なドロワーを実装する。ドキュメントのサンプルコードに TypeScript の型アノテーションを追加したコードが以下になる。[^1]

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

とりあえず動くドロワーを実装できた。起動してみよう。

![InitialVersion](https://raw.githubusercontent.com/ishikawa/my-zenn-content/main/articles/build-slide-menu-with-expo-and-react-navigation/InitialVersion.gif)

しかし、このままでは思い描いた動作とは程遠い。

1. **ドロワーがスクリーンに重なっている**（一緒に移動するようにしたい）
2. **ドロワーの幅が狭すぎる**（もっと広くしたい）
3. **ドロワーの内容が固定**（登録済みの画面一覧が表示されるだけ）

順番に解決していこう。

## ドロワーが開閉するときにスクリーンも動かすようにする

ドロワーの動きは [`drawerType`](https://reactnavigation.org/docs/drawer-navigator#drawertype) プロパティで変更できる。期待する動きにするためには、`drawerType` を `"slide"` に変更する。

```typescript
<NavigationContainer>
  <Drawer.Navigator drawerType="slide">
    ...
  </Drawer.Navigator>
```

早速動かしてみよう。

![SlideDrawer](https://raw.githubusercontent.com/ishikawa/my-zenn-content/main/articles/build-slide-menu-with-expo-and-react-navigation/SlideDrawer.gif)

期待通りに動いているようだ。次はドロワーの幅を広くしたい。

## ドロワーの幅を調整する

今のままだとドロワーの幅が狭すぎるので、もう少し広くしたい。これは [`drawerStyle`](https://reactnavigation.org/docs/drawer-navigator/#drawerstyle) で自由にスタイルを指定できるので、ここで `width` を指定してやればいい。

```typescript
export default function App() {
  return (
    <NavigationContainer>
      <Drawer.Navigator drawerType="slide" drawerStyle={styles.drawer}>
...
const styles = StyleSheet.create({
  ...
  drawer: {
    width: "80%",
  },
});
```

これで幅を広げることもできた。

![DrawerWidth](https://raw.githubusercontent.com/ishikawa/my-zenn-content/main/articles/build-slide-menu-with-expo-and-react-navigation/DrawerWidth.png)

次はいよいよ、ドロワーの中身を独自のビューに置き換えよう。

## ドロワーの中身をカスタマイズ

ドロワーに表示される内容は [`drawerContent`](https://reactnavigation.org/docs/drawer-navigator/#drawercontent) と [`drawerContentOptions`](https://reactnavigation.org/docs/drawer-navigator/#drawercontentoptions) でカスタマイズできる。

`drawerContent` でドロワーの中身として表示されるビューを上書きできるが、デフォルトでは以下のようになっている。

```typescript
import {
  DrawerContentScrollView,
  DrawerItemList,
} from '@react-navigation/drawer';

function CustomDrawerContent(props) {
  return (
    <DrawerContentScrollView {...props}>
      <DrawerItemList {...props} />
    </DrawerContentScrollView>
  );
}
```

デフォルトの表示は `drawerContentOptions` でカスタマイズできる。たとえば、以下のように書き換えてみよう。

```typescript
<NavigationContainer>
  <Drawer.Navigator
    drawerType="slide"
    drawerStyle={styles.drawer}
    drawerContentOptions={{
      activeTintColor: "#e91e63",
      itemStyle: { marginVertical: 30 },
    }}
  >
  ...
```

ドロワーに表示されるメニュー項目の間隔と色が変わったのが分かると思う。他のオプションについては [API リファレンス](https://reactnavigation.org/docs/drawer-navigator/#drawercontentoptions)を参考にしてほしい。

![DrawerOptions](https://raw.githubusercontent.com/ishikawa/my-zenn-content/main/articles/build-slide-menu-with-expo-and-react-navigation/DrawerOptions.png)

もちろん、 `drawerContentOptions` では、そもそもやりたかった「ドロワーの中身を独自のビューで置き換え」は実現できない。それをやるには `drawerContent` を使って、中身を丸ごと置き換えてやる必要がある。

`drawerContent` にはドロワーの中身となる React コンポーネントを返す関数を指定する。以下のような感じだ。

```typescript
export default function App() {
  return (
    <NavigationContainer>
      <Drawer.Navigator
        drawerType="slide"
        drawerStyle={styles.drawer}
        drawerContent={() => <View></View>}
      >
...
```

また、この関数には navigation オブジェクトを含む props が渡される。

```typescript
export default function App() {
  return (
    <NavigationContainer>
      <Drawer.Navigator
        drawerType="slide"
        drawerStyle={styles.drawer}
        drawerContent={props => <DrawerMenu {...props} />}
      >
...
```

ここでは、サンプルとして、アイコン画像とメニュー項目を配置して、それっぽいビューを作ってみる。ここで気をつけたいのは、ドロワーに表示するビューに渡される props の型が `DrawerContentComponentProps` になることくらいだろうか。

```typescript
import React from "react";
import {
  StyleSheet,
  Button,
  Text,
  Image,
  View,
  TouchableOpacity,
} from "react-native";
import {
  createDrawerNavigator,
  DrawerNavigationProp,
  DrawerContentComponentProps,
} from "@react-navigation/drawer";
import { NavigationContainer } from "@react-navigation/native";
import { StatusBar } from "expo-status-bar";
import { Ionicons } from "@expo/vector-icons";

// @ts-ignore
import picture from "./assets/picture.png";
...
function DrawerMenu({ navigation }: DrawerContentComponentProps) {
  return (
    <View style={styles.menuContainer}>
      <Image source={picture} resizeMode="contain" style={styles.picture} />
      <View>
        <Text style={styles.name}>takanori_is</Text>
        <Text style={styles.nickname}>@takanori_is</Text>
      </View>
      <TouchableOpacity
        style={styles.menuItem}
        onPress={() => {
          navigation.navigate("Home");
        }}
      >
        <Ionicons name="home-outline" size={24} />
        <Text style={styles.menuItemLabel}>ホーム</Text>
      </TouchableOpacity>
      <TouchableOpacity
        style={styles.menuItem}
        onPress={() => {
          navigation.navigate("Notifications");
        }}
      >
        <Ionicons name="notifications-outline" size={24} />
        <Text style={styles.menuItemLabel}>通知</Text>
      </TouchableOpacity>
    </View>
  );
}
...
const styles = StyleSheet.create({
  ...
  menuContainer: { paddingVertical: 70, paddingHorizontal: 20 },
  picture: {
    width: 64,
    height: 64,
    borderRadius: 64 / 2,
    marginBottom: 7,
  },
  name: { fontSize: 18, fontWeight: "bold" },
  nickname: { fontSize: 15, color: "gray" },
  menuItem: { flexDirection: "row", marginTop: 24 },
  menuItemLabel: { marginLeft: 15, fontSize: 20, fontWeight: "300" },
});
```

`expo start` してシミュレーターで動作確認してみよう。たしかに、アイコン画像とメニュー項目を持つ独自のビューが配置されている。

![CustomDrawer](https://raw.githubusercontent.com/ishikawa/my-zenn-content/main/articles/build-slide-menu-with-expo-and-react-navigation/CustomDrawer2.gif)



[^1]: React Navigation の TypeScript による型づけについては [Type checking with TypeScript | React Navigation](https://reactnavigation.org/docs/typescript) を参考