---
title: "Babel プラグインを書いて React Hooks 時代に追いつく"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "babel"]
published: false
---

_この記事は [ABEJA Advent Calendar 2020](https://qiita.com/advent-calendar/2020/abeja) の 8 日目の記事です。_

どうも、こんにちは。株式会社 ABEJA の石川です。今年興奮した技術的トピックは [Erlang VM の JIT コンパイラ](https://github.com/erlang/otp/pull/2745) と Apple の M1 チップです。よろしくお願いします。**MacBook Air は最高の体験ですね。**

![](https://github.com/ishikawa/my-zenn-content/blob/main/articles/convert-legacy-code-with-home-brewed-babel-plugin/MacBookAir.jpg?raw=true)

去年[^1]と一昨年[^2]は、機械学習プラットフォーム [ABEJA Platform](https://developers.abeja.io/) の使い方について書きましたが、今年は少し趣向を変えて、開発の舞台裏について書いてみたいと思います。

もともと、言語処理系やコンパイラに興味がある[^3]のですが、最近ではこの趣味を実務に活かすべく、**React アプリケーションの自動コード変換**に取り組んでいる、というお話です。

## これまでの ABEJA Platform の Web フロントエンドは...

ABEJA Platform の Web フロントエンドは 2017 年 9 月から開発をはじめました。

DOM のレンダリングに React を採用し、ステート管理には Redux と非同期通信のミドルウェアに Redux-Observable、Babel によるトランスパイルと Flow による型チェック、バンドラーに Webpack を採用していました。途中、プロトタイプ時に使っていた Material UI から Semantic UI への変更はあったものの、今でもこの構成は大きくは変わっていません。

![](https://github.com/ishikawa/my-zenn-content/blob/main/articles/convert-legacy-code-with-home-brewed-babel-plugin/ABEJAPlatform.png?raw=true)

当時の React のバージョンは 15.5.4。`React.createClass` が非推奨となり、React コンポーネントの開発に ES2015 のクラスが使えるようになりました。[^4]また、公式ブログでは[以下のようにアナウンスされていた](https://ja.reactjs.org/blog/2017/04/07/react-v15.5.0.html)ようです。

> Along with function components, JavaScript classes are now the preferred way to create components in React.

しかし、時代は移ろい、昨年の 2 月にリリースされた [React 16.8](https://reactjs.org/blog/2019/02/06/react-v16.8.0.html) から、世はまさに **React Hooks 時代**です。Hooks の導入以降、クラスによるコンポーネントの実装は第一選択とはならず、関数コンポーネントと Hooks の利用が主流になっています。

ABEJA Platform でも、昨年の 3 月末に React 16.8.6 に更新し、Hooks が使える状態になっていましたが、その利用はせいぜい、

- 新しく書くコンポーネントは、できるだけ関数コンポーネントで実装する
- Hooks は組み込みの `useState()` と `useEffect()` のみ

程度に収まっていました。

しかし、[React 17.0 のアナウンス](https://reactjs.org/blog/2020/10/20/react-v17.html)をきっかけに、フロントエンドの技術スタックをアップデートする気運が（個人的に）高まり、以下のようなアップデートを進めています。

1. React を含む各種ライブラリのアップデート
2. 既存コードを関数コンポーネントに書き換える
3. 各種ライブラリが提供する Hooks を積極的に使う
4. Flow をやめて TypeScript に移行する

現在は上記のうち、4 の調査と移行を試しながら、3 を進めている状態ですが、既存コードの変換は手作業ではなく、**Babel プラグインによる自動変換**というアプローチを採用しています。

この記事の目的は、このコード変換について紹介することですが、まずはどうして関数コンポーネントと Hooks に移行するのかを理解してもらうため、これらの利点と背景について説明します。

### 関数コンポーネントと Hooks

公式ドキュメント[^5]でも解説されている通り、関数コンポーネントと Hooks には以下の利点があります。

1. **再利用性** コンポーネントの階層構造を壊すことなく、ロジックを再利用できる
2. **コードの見通しの良さ** ライフサイクルメソッドに様々なロジックを詰め込むのではなく、Effect hook で関連するロジックだけをまとめることができる
3. **学習の容易さ** 関数と比較して、クラスは書くのも読んで理解するのも難しい
4. **コンパイラによる最適化のしやすさ**  JavaScript のクラスはその特性上、コンパイラによる最適化が難しく、コード圧縮やホットコードリローディングとの相性も悪い

それぞれについては、上記ドキュメントで詳しく解説されてるので、ここで繰り返すつもりはありません。いくつか興味深い点についてのみ指摘しておくに留めます。

#### HOC (High-Order Components) の排除

Hooks 以前にコンポーネントで共通ロジック (横断的関心) を再利用するためには [HOC (High-Order Components)](https://reactjs.org/docs/higher-order-components.html) を使うのが一般的でした。React Redux や React Router といった周辺ライブラリも HOC を提供しており、実際に、ABEJA Platform でもこれらの HOC を利用しています。

```javascript
export default withRouter(
  connect(
    mapStateToProps,
    mapDispatchToProps
  )(injectIntl(MyComponent))
);
```

この例では、React Router の `withRouter` React Redux の `connect` react-intl の `injectIntl` を使っています。更に、フォームのある画面では Redux form の `reduxForm` を利用するため以下のようになります。

```javascript
export default withRouter(
  connect(
    mapStateToProps,
    mapDispatchToProps
  )(injectIntl(reduxForm(formOpts)(MyComponent)))
);
```

HOC はコンポーネントの階層構造を深くするため、デバッグが面倒になることは、上記の React 公式ドキュメントでも指摘されてますが、他にも頭の痛い問題として **TypeScript による型付けが面倒くさい**、というのがあります。

というのも、HOC はコンポーネントの props に新たなプロパティを追加することが多く、それらを毎回型定義してあげる必要があります。

React Redux の `connect` について考えてみましょう。

connect HOC を利用するコンポーネントの Props を、愚直に手作業で型付けしてみます（コード例は [React Redux の公式ドキュメント](https://react-redux.js.org/using-react-redux/static-typing)からの転載）。

```typescript
import { connect } from 'react-redux'

interface StateProps {
  isOn: boolean
}

interface DispatchProps {
  toggleOn: () => void
}

interface OwnProps {
  backgroundColor: string
}

type Props = StateProps & DispatchProps & OwnProps
```

`connect` HOC でラップされたコンポーネントに与えられる props は

- コンポーネント固有の props `OwnProps`
- `mapStateToProps` から発生する props `StateProps`

- `mapDispatchToProps` から発生する props `DipatchProps`

の 3 つで構成されており、これらを TypeScript の [Intersection Types](https://www.typescriptlang.org/docs/handbook/unions-and-intersections.html#intersection-types) を用いて合成します。

バージョン 7.12 からは [`ConnectedProps` 型を使うことで若干楽になりました](https://react-redux.js.org/using-react-redux/static-typing#inferring-the-connected-props-automatically)が、これはこれで今までの `connect` に慣れた身からすると煩雑さと違和感を感じます。

一方、Hooks の場合は HOC とは異なり props を追加するわけではないのと、TypeScript の型推論もあるため、多くの場合、冗長な型定義は必要ありません。

```typescript
interface RootState {
  isOn: boolean
}

// TS infers type: (state: RootState) => boolean
const selectIsOn = (state: RootState) => state.isOn

// TS infers `isOn` is boolean
const isOn = useSelector(selectIsOn)
```

実際に React Redux のドキュメントでも [Hooks API の方が簡単だと紹介されています](https://react-redux.js.org/using-react-redux/static-typing#recommendations)。

ABEJA Platform のコードでも現状、Flow を用いて型付けをしているため同様の冗長な型定義が必要になっている[^6]のですが、TypeScript に移行するときは Hooks を利用して簡潔なコードにしたいところです。

他の代表的なライブラリでも、従来の HOC の代替として Hooks が提供されています。

- React Router はバージョン 5.1 から、[`withRouter`](https://reactrouter.com/web/api/withRouter) HOC の代替として [`useHistory`](https://reactrouter.com/web/api/Hooks/usehistory), [`useLocation`](https://reactrouter.com/web/api/Hooks/uselocation), [`useParams`](https://reactrouter.com/web/api/Hooks/useparams), [`useRouteMatch`](https://reactrouter.com/web/api/Hooks/useroutematch) Hooks が提供されるようになりました [^7]
- react-intl はバージョン 3 から [`injectIntl`](https://formatjs.io/docs/react-intl/api#injectintl-hoc) HOC の代替として、[`useIntl`](https://formatjs.io/docs/react-intl/api#useintl-hook) Hooks が提供されるようになりました。[^8] また、このバージョンから TypeScript で書き直されました[^9]

:::message

残念ながら、Redux Form の開発はメンテナンスのみになっている[^10]ため、議論はされていました[^11]が現時点で Hooks の提供はありません

:::

Hooks を使うことで、開発者はコンポーネントの階層構造を深くすることなく、型付けも容易な関数でコンポーネントを記述できるようになっています。[^12]

#### コンパイラによる最適化のしやすさ

現代ではコンパイラ技術が、言語処理系以外のさまざまな局面で活用されるようになっています。[^13]

もちろん、フロントエンド開発も例外ではなく、webpack などのバンドラーが不要なコードを削除する [Tree Shaking](https://webpack.js.org/guides/tree-shaking/) は JavaScript における [Dead-Code Elimination](https://en.wikipedia.org/wiki/Dead_code_elimination) ですし、React の [JSX](https://reactjs.org/docs/jsx-in-depth.html) は `React.createElement(...)` の [Syntax Sugar](https://en.wikipedia.org/wiki/Syntactic_sugar) です。[^14]

昨今のフロントエンドフレームワークでも、コンパイラ技術は取り入れられています。[Anglar](https://angular.io/) 2 以降では[ AOT コンパイルの導入](https://angular.io/guide/aot-compiler)により、テンプレートの事前コンパイルと[定数畳み込み](https://en.wikipedia.org/wiki/Constant_folding)による最適化が可能です。

[Svelte](https://svelte.dev/) フレームワークではこれを更に進めて、AOT コンパイルされた結果には Svelte のランタイムは不要であり、React のような[変更検知のための複雑なアルゴリズム](https://reactjs.org/docs/reconciliation.html)も必要ありません。[^15]

Facebook のプロジェクトで、JavaScript を AOT コンパイルして最適化を行う [Prepack](https://prepack.io/frequently-asked-questions.html) が数年前に話題になりましたが、その流れで React コンポーネントの畳み込み、インライン化も検討されていたようです。[^16]

これは、たとえば、以下のようなコンポーネント `Bar` があったときに:

```javascript:Foo.js
function Foo(props) {
  if (props.data.type === 'img') {
    return <img src={props.data.src} className={props.className} alt={props.alt} />;
  }
  return <span>{props.data.type}</span>;
}
Foo.defaultProps = {
  alt: "An image of Foo."
};
```

```javascript:Classes.js
var CSSClasses = {
  bar: 'bar'
};
module.exports = CSSClasses;
```

```javascript:Bar.js
var Foo = require('Foo');
var Classes = require('Classes');
function Bar(props) {
  return <Foo data={{ type: 'img', src: props.src }} className={Classes.bar} />;
}
```

以下のように変換してしまおう、というものです。

```javascript:Bar.js
function Bar_optimized(props) {
  return <img src={props.src} className="Bar" alt="An image of Foo." />;
}
```

もちろん、これは単純に置き換えるだけでは駄目で、[エスケープ解析](https://en.wikipedia.org/wiki/Escape_analysis)などで `CSSClasses` と `Foo.defaultProps` が変更されないことを保証する必要があります。しかも、これが関数コンポーネントではなく、より複雑なセマンティクスを持つクラスコンポーネントでは、変換が難しくなるのは明白でしょう。

今後、AOT コンパイルを含むコンパイラ最適化が React に導入された場合も、変換できない (あるいは、変換可能かどうかが簡単には判別できない) ケースでは、最適化が施されずに「遅い」コードパスが実行されることが予想できます。このことを考慮しても、コンポーネントはクラスよりも関数で実装した方が将来の最適化の恩恵を受けられる可能性が高まります。

## Hooks に乗り換える

さて、長い前置きはここまでです。

先に説明した通り、現在は Hooks への移行のために、

- 既存コードを関数コンポーネントに書き換え
- 各種ライブラリが提供する Hooks を積極的に使う

ように、コードベースを変更しているところです。また、変更は手作業ではなく、**Babel プラグインによる自動変換**というアプローチを採用しています。

以降では「react-intl の `injectIntl` HOC を `useIntl` Hooks に置き換える変換」を例に、具体的な変換処理の説明と何故このようなアプローチを採用したかの説明もしたいと思います。

### react-intl の `injectIntl` HOC を `useIntl` Hooks に置き換える

Hooks を利用できるのは関数コンポーネントのみであるため、まず、同様の自動変換のアプローチで、**既存コードを関数コンポーネントに書き換え**ました。当然、変換できないケースもありましたが、ここで約 100 個のクラスコンポーネントを関数コンポーネントに書き換えました。

ただ、関数コンポーネントへの変換処理は複雑なのと、ABEJA Platform のコードベースに依存した変換が大分め、ブログで紹介しても散漫な内容になりそうです。今回は分かりやすい例として **react-intl の `injectIntl` HOC を `useIntl` Hooks に置き換える変換**を紹介します。

これは簡単に言えば、以下のような JavaScript コードがあったとして、

```javascript
import { injectIntl } from 'react-intl';

function MyComponent(props: Props) {
  const {
    intl: { formatMessage }
  } = props;

  ...
}

export default injectIntl(MyComponent);
```

このコードを以下のように変換します。

```javascript
import { useIntl } from 'react-intl';

function MyComponent(props: Props) {
  const { formatMessage } = useIntl();
  ...
}

export default MyComponent;
```

パッと見た感じ、さほど難しい変換ではありません。さあ、頑張って変換していきましょう。

### 自動変換の方針

既存のコードベースで `injectIntl` を使っている箇所を検索したところ、176 個のファイルが対象になりました。

![](https://raw.githubusercontent.com/ishikawa/my-zenn-content/main/articles/convert-legacy-code-with-home-brewed-babel-plugin/SearchInEditor.png)

それほど難しくない変換とはいえ、この量の変更を手作業で行うのは面倒ですし、ケアレスミスも発生しそうです。また、ひとりで開発しているわけではないので、変更作業中に他のメンバーがマージした変更とコンフリクトするかもしれません。

そうすると、変換処理は自動化し、

- 大量のファイルに一気に適用でき、
- 差分がコンフリクトする場合は、簡単にやり直せる

ようにしておくのが良さそうです。なによりプログラマなので**退屈な単純作業には堪えられない**、という理由が大きいです。

#### 正規表現による置換

では、自動化の方針ですが、すぐに思いつくのは**正規表現による置換**です。

たとえば、`injectIntl(MyComponent)` を `MyComponent` に置き換えるには、

```perl
s/injectIntl\((\w+)\)/$1/
```

まあ、こんな正規表現で行けそうですし、新しめの正規表現エンジンであれば再帰パターンも扱えるのでネストした構造を含むプログラムの変換もそれなりにできたりします。[^17]

しかし、文字列中の扱いなど対応できないエッジケースは存在しますし、空白・改行の無視や共通するパターンを変数化したりすると、あっというまにメンテナンス不能なコードになってしまいます (読むのも書くのも苦痛)。

やはり、ここは、変換対象の JavaScript コードを一度、**抽象構文木 (Abstract Syntax Tree; AST) に変換しましょう。**

たとえば以下のような JavaScript コードを AST に変換すると、

```javascript
function add(a, b) {
  return a + b;
}
add(1, 2);
```

次のような AST になります。[^18]

![](https://github.com/ishikawa/my-zenn-content/blob/main/articles/convert-legacy-code-with-home-brewed-babel-plugin/AST.png?raw=true)

AST に対して必要な変換を施した上で、結果の JavaScript コードを出力するプログラムを書けば、煩雑な正規表現に悩まされることもなく、メンテナンス可能な自動化ができそうです。[^19]

ABEJA Platform では既に Babel を導入済みであったことや、ある程度まとまったドキュメントが用意されていたことから、今回は Babel プラグインとして変換処理を実装することにしました。

## Babel プラグインを書く

ここからは実際のコードを紹介しながら、どのように react-intl の `injectIntl` HOC を `useIntl` Hooks に置き換えるかを説明していきます。

一般的な Babel プラグインの書き方や導入については、すでに有用な情報がインターネットで公開されているため、ここでは立ち入りません。今回参照した中では [Babel Plugin Handbook](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/en/plugin-handbook.md) が情報も正確でもっとも役に立ちました。

### ガイドライン

プラグインの書き方以外で、個人的に特筆しておきたい点がいくつかあります。

#### TypeScript で書く

すでにデファクト・スタンダードになりつつある TypeScript ですが、Babel プラグインも TypeScript で、そして VSCode で書くことをおすすめします。

[Babel プラグインを TypeScript で開発する](https://zenn.dev/takanori_is/articles/babel-plugin-development-with-typescript)でも紹介したのですが、[Type predicates](https://www.typescriptlang.org/docs/handbook/advanced-types.html#using-type-predicates) によって適切に型推論されますし、メソッドの引数以外で明示的な型アノテーションが必要なケースはほとんどありません。

```typescript
if (
  t.isCallExpression(node.expression) &&  // node.expression: t.Expression
  t.isSuper(node.expression.callee) &&    // node.expression: t.CallExpression
  node.expression.arguments.length === 1 &&
  t.isIdentifier(node.expression.arguments[0], { name: 'props' })
) {
  // ...
}
```

また、プラグインの開発中は [@babel/types](https://babeljs.io/docs/en/babel-types) のメソッドを何度も呼ぶことになりますが、これだけ大量のメソッドがあると VSCode による補完がないとやってられません。

#### AST Explorer は友達

[AST Explorer](https://astexplorer.net/) を使うと、JavaScript コードの AST 表現を即座に確認できます。プラグイン開発においてはなくてはならないツールなので、ぜひ使い方を覚えて、いつでも参照できるようにしておきましょう。

![ASTExplorer](https://github.com/ishikawa/my-zenn-content/blob/main/articles/convert-legacy-code-with-home-brewed-babel-plugin/ASTExplorer.png?raw=true)

#### 無理して汎用的なプラグインを書こうとしない

AST の変換はあらゆるケースに対応しようとすると、思った以上に大変です。たとえば、先程の正規表現の話のときにも出した、

```javascript
export default injectIntl(MyComponent);
```

を

```javascript
export default MyComponent;
```

変換するケースを考えてみましょう。これを Babel プラグインで実装するのは...そう、簡単ですよね？　以下のような Visitor を書けば対応できそうです。

```typescript
const removeInjectIntlVisitor = {
  CallExpression(path: NodePath<t.CallExpression>) {
    const { node } = path;

    if (t.isIdentifier(node.callee, { name: "injectIntl" })) {
      path.replaceWith(node.arguments[0]);
    }
  },
};
```

しかし、本当にこれで十分でしょうか？　少し考えてみただけでも、以下のケースを考慮する必要がありそうです。

- `injectIntl` HOC が、別名でインポートされていないか？
- `injectIntl` という識別子がインポートされた HOC ではなく、他の変数ではないか？
- 対象の CallExpression ノードが `export default` されているか？

これらのケースをすべて洗い出して対応していくのはかなり大変です。

しかし、多くのコードベースにはある程度の秩序があり、従っている慣習があります。これらの知見を前提にして、自分たちのコードベースのみに対応できるプラグインを書きましょう。

実際、今回の変換対象であるコードベースでは、`injectIntl` HOC が、別名でインポートされることはないですし、`injectIntl(...)` という呼び出しが、`export default` されるところ以外で現れることもありません。なので、ほぼ上記と同じコードで変換を実装していますし、それで何の問題もありませんでした。

#### Jest の Snapshot テストを活用しよう

AST の変換では、あるパターンの変更が別のパターンに影響を与えることがあり、少し前まではうまくいっていた変換がいつのまにか動かなくなっていることがザラにあります。

自動テストを必ず書きましょう。

しかし、テスト対象のコードを AST に変換して、期待するノードを逐一チェックする、そんなテストを書くのは嫌ですよね？　テスト自体のメンテナンス性も低そうです。ぜひ、[Jest の Snapshot テスト](https://jestjs.io/docs/en/snapshot-testing)を使いましょう。

今回のような用途で Babel プラグインを書くときは、以下のような開発フローが定番になります。

1. 変換対象のコードを Fixture として用意する
2. AST Explorer の出力を参考にしながら、変換処理を実装する
3. `npx babel` で変換結果を確認する
4. `npx jest` で過去の Snapshot テストが通るか確認する
5. うまく変換できるまで、2 から 4 を繰り返す
6. 最後に `npx jest -u` で Snapshot を更新する

#### スコープの情報が更新されていないときは crawl() する

Babel にはローカル変数の束縛をチェックしたり、ユニークな変数名を生成したりできる便利な機能として [Scope](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/en/plugin-handbook.md#toc-scope) が用意されていますが、AST を直接プロパティを変更した場合などに情報が反映されない場合があります。

そのようなときは `Scope.crawl()` メソッドで更新しましょう。

```javascript
path.scope.crawl();
```

### injectIntl の変換

では、コードを見ながら、いくつか重要な箇所を解説していきます。まず、これがメインの Visitor になります。

```typescript
const visitor = {
  Program(path: NodePath<t.Program>) {
    const state = {
      foundClassComponent: false,
      importedInjectIntlName: null,
      needsUseIntl: false,
    };

    path.traverse(reactClassComponentVisitor, state);
    if (state.foundClassComponent) {
      throw new Error("Unsupported: class component");
    }

    path.traverse(replaceWithUseIntlVisitor, state);

    path.traverse(importInjectIntlVisitor, state);
    if (!state.importedInjectIntlName) {
      throw new Error("import { injectIntl } not found.");
    }

    path.traverse(removeIntlShapeTypeVisitor, state);
    path.traverse(removeIntlFromPropsTypeVisitor, state);
    path.traverse(removeInjectIntlVisitor, state);
    path.traverse(removeEmptyObjectPatternVarDecl, state);
  },
};
```

見ての通り、特に難しいところはなく、

- 変換処理は複数の Visitor に分割して実装
- すべての Visitor で共有する state を作って、情報の受け渡しはそこで行う
- 変換処理をやめたいときは、単純に Error を投げる

このような感じです。

それでは、各 Visitor について、いくつか見ていきましょう。

#### reactClassComponentVisitor

再三書いているとおり、Hooks は関数コンポーネントでしか使えないので、変換対象は関数コンポーネントだけに絞る必要があります。しかし、関数コンポーネントかどうかを判別するのは割と面倒なので、ここでは「クラスコンポーネントでなければ関数コンポーネント」ということにしています。

クラスコンポーネントかどうかは、これも単純に「React.Component を継承したクラスがあるかどうか」で判定しています。

```typescript
// We must not convert class component.
const reactClassComponentVisitor = {
  ClassDeclaration(path: NodePath<t.ClassDeclaration>, state: VisitorState) {
    const { node } = path;

    // class Foo extends React.Component {...}
    if (
      t.isMemberExpression(node.superClass) &&
      t.isIdentifier(node.superClass.object, { name: "React" }) &&
      t.isIdentifier(node.superClass.property, { name: "Component" })
    ) {
      state.foundClassComponent = true;
    }
  },
};
```



もちろん、変換対象のファイルには React コンポーネント以外も含まれているわけですが、その場合は後続の importInjectIntlVisitor で「`injectIntl` をインポートしているか？」をチェックしているので問題ありません。

#### replaceWithUseIntlVisitor

早くもメインディッシュが出てきました。

```javascript
const {
  ...
  intl: { formatMessage }
} = props;
```

のようなコードを

```javascript
const {...} = props;
const { formatMessage } = useIntl();
```

に変換する Visitor です。

まずは全体のコードを載せます。多少分量が多いのですが、基本的には愚直に変換をしているだけです。

```typescript
// Transform:
//
// ```javascript
// const {
//   ...,
//   intl: { ... }
// } = props;
// ```
//
// to:
//
// ```javascript
// const {...} = props;
// const {...} = useIntl();
// ```
const replaceWithUseIntlVisitor = {
  VariableDeclaration(
    path: NodePath<t.VariableDeclaration>,
    state: VisitorState
  ) {
    const { node } = path;
    let rewriteBinding = false;

    for (const decl of node.declarations) {
      // {...} =
      if (!t.isObjectPattern(decl.id)) {
        continue;
      }
      if (
        // {...} = props;
        !(
          t.isIdentifier(decl.init, { name: "props" }) ||
          // {...} = this.props;
          (t.isMemberExpression(decl.init) &&
            t.isThisExpression(decl.init.object) &&
            t.isIdentifier(decl.init.property, { name: "props" }))
        )
      ) {
        continue;
      }

      const props: (t.RestElement | t.ObjectProperty)[] = [];

      for (const prop of decl.id.properties) {
        if (
          t.isObjectProperty(prop) &&
          t.isIdentifier(prop.key, { name: "intl" })
        ) {
          if (!(t.isIdentifier(prop.value) || t.isObjectPattern(prop.value))) {
            throw new Error(
              `Expected: { intl: {...} } or { intl } at line:${node.loc?.start.line}`
            );
          }

          // Insert: const {...} = useIntl();
          path.insertAfter([
            t.variableDeclaration("const", [
              t.variableDeclarator(
                prop.value,
                t.callExpression(t.identifier("useIntl"), [])
              ),
            ]),
          ]);
          state.needsUseIntl = true;
          rewriteBinding = true;

          continue;
        }
        props.push(prop);
      }

      decl.id.properties = props;
    }

    ...
  },
};
```

見たままですが、VariableDeclaration をループしながら変換できる対象を見つけたら変換しているだけです。ただ、ループを抜けたあとに追加で以下のような変換を施しています。

```typescript
// props: { intl: IntlShape }
const binding = path.scope.getBinding("props");
if (
  rewriteBinding &&
  binding &&
  t.isIdentifier(binding.identifier) &&
  t.isTypeAnnotation(binding.identifier.typeAnnotation) &&
  t.isObjectTypeAnnotation(binding.identifier.typeAnnotation.typeAnnotation)
) {
  const newProps = [];
  const ty = binding.identifier.typeAnnotation.typeAnnotation;
  for (const prop of ty.properties) {
    if (
      t.isObjectTypeProperty(prop) &&
      t.isIdentifier(prop.key, { name: "intl" }) &&
      t.isGenericTypeAnnotation(prop.value) &&
      t.isIdentifier(prop.value.id, { name: "IntlShape" })
    ) {
      continue;
    }
    newProps.push(prop);
  }
  ty.properties = newProps;
}
```

今回のコードベースは Flow で型付けされており、**AST を変換するときは Flow の型チェックも通るように実装しなくてはなりません。**

最後のこの変換は、Babel の Scope を利用して、

```javascript
function MyComponent(props: { intl: IntlShape }) {...}
```

のように型付けされた JavaScript コードから `intl` を取り除いています。

#### importInjectIntlVisitor

この Visitor では react-intl からのインポートを探し、

- `injectIntl` があれば削除
- `useIntl` が必要であれば追加
- 結果的にインポートするシンボルがなければ、import 文自体を削除

しています。同一モジュールからの複数インポートは ESLint で禁止しているので、既存の import 文を変更することで対応しています。

```typescript
const importInjectIntlVisitor = {
  ImportDeclaration(path: NodePath<t.ImportDeclaration>, state: VisitorState) {
    const { node } = path;

    if (
      !(
        node.importKind === "value" &&
        t.isStringLiteral(node.source, { value: "react-intl" })
      )
    ) {
      return;
    }

    const specifiers: typeof node.specifiers = [];

    for (const specifier of node.specifiers) {
      if (
        t.isImportSpecifier(specifier) &&
        t.isIdentifier(specifier.imported, { name: "injectIntl" })
      ) {
        state.importedInjectIntlName = specifier.local.name;

        if (state.needsUseIntl) {
          specifier.imported = t.identifier("useIntl");
          specifier.local = t.identifier("useIntl");
        } else {
          continue;
        }
      }
      specifiers.push(specifier);
    }

    if (specifiers.length === 0) {
      path.remove();
    } else {
      node.specifiers = specifiers;
    }
  },
};
```

#### removeEmptyObjectPatternVarDecl

途中の Visitor はだいたい同じ処理ばかりなので、一気に最後の removeEmptyObjectPatternVarDecl を見てみましょう。

[replaceWithUseIntlVisitor](#replaceWithUseIntlVisitor) で props を置き換えた結果、props が空になることがあります。

```javascript
function MyComponent(props) {
  const {} = props;
  const { intl } = useIntl();
  ...
}
```

空の ObjectPattern および、未使用の変数も ESLint で禁止されているため、この Visitor を使って以下のようなコードに変換します。

```JavaScript
function MyComponent(_props) {
  const { intl } = useIntl();
  ...
}
```

実装は以下の通り。ここでも Scope が大活躍しています。

```typescript
// Remove a variable declaration which contains empty object pattern only,
// bacause ESLint complains about it.
const removeEmptyObjectPatternVarDecl = {
  VariableDeclarator(
    path: NodePath<t.VariableDeclarator>,
    _state: VisitorState
  ) {
    const { node } = path;

    if (t.isObjectPattern(node.id) && node.id.properties.length === 0) {
      path.remove();

      // Rename unused variables (in function arguments), bacause ESLint complains about it.
      const binding = path.scope.getBinding("props");

      if (binding && !binding.referenced) {
        path.scope.rename("props", "_props");
      }
    }
  },
};
```

## まとめ

最終的に完成したプラグインで、見事 147 個のファイルを変換できました。

![](https://github.com/ishikawa/my-zenn-content/blob/main/articles/convert-legacy-code-with-home-brewed-babel-plugin/Merged.png?raw=true)

エディタの検索結果では対象は 176 個だったので、変換できなかったパターンが 30 個ほどあったことになりますが、手作業のつらさを思えば十分満足のいく結果です。

実際のコードを見ていただいたことで、**汎用的なものを目指さなければ Babel プラグインによる変換は思ったより簡単**、ということが幸いです。少なくとも、すでに強固なインフラストラクチャーが整備されている Babel は、正規表現で書いたアドホックなスクリプトより何倍も書きやすく読みやすいのは間違いありません。

また、VSCode による TypeScript のサポートも強力で、非常に快適な開発環境でした。ますます、TypeScript に移行したくなりました。

[^1]: [ABEJA Platform + LINE Botで機械学習アプリをつくる - Qiita](https://qiita.com/ishikawa@github/items/3b27d7836032f0e42f88)
[^2]: [ABEJA Platform の認証についてまとめる - Qiita](https://qiita.com/ishikawa@github/items/150a0705d9581c1000c6)
[^3]: [builderscon tokyo 2019 - Elixir: Under the Hood - Qiita](https://qiita.com/ishikawa@github/items/3d054f3d29920107687f)
[^4]: React の歴史については [React今昔物語 - ICS MEDIA](https://ics.media/entry/200310/) が大変よくまとまっています。
[^5]: [Introducing Hooks – React](https://reactjs.org/docs/hooks-intro.html#motivation)
[^6]: なお、Flow の場合、connect 自体の型付けも難しいです。[flow-notes/annotating-connected-components.md at master · wgao19/flow-notes](https://github.com/wgao19/flow-notes/blob/master/react/annotating-connected-components.md)
[^7]: [React Training: React Router v5.1](https://reacttraining.com/blog/react-router-v5-1/)
[^8]: [New useIntl hook as an alternative of injectIntl HOC / Upgrade Guide (v2 -> v3) | Format.JS](https://formatjs.io/docs/react-intl/upgrade-guide-3x/#new-useintl-hook-as-an-alternative-of-injectintl-hoc)
[^9]: [TypeScript Support / Upgrade Guide (v2 -> v3) | Format.JS](https://formatjs.io/docs/react-intl/upgrade-guide-3x/#typescript-support)
[^10]: [Roadmap - v9 · Issue #4467 · redux-form/redux-form](https://github.com/redux-form/redux-form/issues/4467)
[^11]: [Provide React Hooks · Issue #4317 · redux-form/redux-form](https://github.com/redux-form/redux-form/issues/4317)
[^12]: ただし、Hooks にも注意すべき点はあり、依然、HOC にもいくつかの利点があります。興味のある方は、[React Boston 2019: Hooks, HOCs, and Tradeoffs · Mark's Dev Blog](https://blog.isquaredsoftware.com/2019/09/presentation-hooks-hocs-tradeoffs/) を参考にしてください 
[^13]: 正規表現の JIT コンパイル、深層ニューラルネットワークの最適化、など
[^14]: React 17.0 からは若干異なります。[Introducing the New JSX Transform – React Blog](https://reactjs.org/blog/2020/09/22/introducing-the-new-jsx-transform.html)
[^15]: [Virtual DOM is pure overhead](https://svelte.dev/blog/virtual-dom-is-pure-overhead)
[^16]: [Optimizing Compiler: Component Folding · Issue #7323 · facebook/react](https://github.com/facebook/react/issues/7323)
[^17]: Python では [regex](https://pypi.org/project/regex/) パッケージで再帰パターンがサポートされています
[^18]: [JointJS - JavaScript diagramming library - Demos.](https://resources.jointjs.com/demos/javascript-ast)
[^19]: Facebook も既存コードベースの変換や、ユーザーが React をアップグレードするときの補助ツールとして、同様の手法でコードを変換するツールを提供しています。[Effective JavaScript Codemods. Tool assisted code modifications can… | by Christoph Nakazawa | Medium](https://medium.com/@cpojer/effective-javascript-codemods-5a6686bb46fb)