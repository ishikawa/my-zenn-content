---
title: "React Testing Library の render で独自のオプションを渡す"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "react", "test"]
published: true
---

今のプロジェクトで React Component のテストを [Enzyme](https://enzymejs.github.io/enzyme/) から [React Testing Library](https://testing-library.com/docs/react-testing-library/intro/) に移行する、という~~苦行~~刺激的なチャレンジをしているので、気づいたことがあればその都度書き残していきたい。

- [Enzyme](https://enzymejs.github.io/enzyme/) は Component に実装に依存しているため、メンテが大変
- 一方、[React Testing Library](https://testing-library.com/docs/react-testing-library/intro/) は Render 結果である DOM 構造をテストするので分かりやすい。実際のユーザー行動に近い
- しかし、異なるコンセプトのもとに書かれたコードを移行するので、「やりたいこと」に対するアプローチの違いをいちいち変換してあげる必要がある。
- コンポーネントのインスタンスメソッドを Mock しているようなテストだと対象コンポーネントの作り自体を変えないと無理っぽい

## Custom Render で Wrapper Component にオプションを渡したい

移行の中で最初に困ったのがこれ。

React Testing Library では、[公式ドキュメントに記載されている](https://testing-library.com/docs/react-testing-library/setup/#custom-render)ように、独自の render メソッドを実装して、対象のコンポーネントを Context Provider でラップできる。たとえば、以下のように Redux や React Router を設定できる。

```javascript
import React from 'react'
import { render } from '@testing-library/react'
import { Router } from 'react-router-dom';
import { createMemoryHistory, MemoryHistory } from 'history';
import configureStore from 'redux-mock-store';
import { Provider } from 'react-redux';

const wrapper = ({ children }) => {
  const mockStore = configureStore();
  const store = mockStore({
        session: {},
        books: {},
      });
  const history = createMemoryHistory();

  return (
    <Provider store={store}>
      <Router history={history}>
        {children}
      </Router>
    </Provider>
  );
};

const customRender = (ui, options) =>
  render(ui, { wrapper, ...options })

// re-export everything
export * from '@testing-library/react'

// override render method
export { customRender as render }
```

このカスタム render を使う側では、特にコードを変えずに import を変えるだけでいい。

```javascript
import { render } from '../test-utils';
...
render(<MyComponent {...props}>);
```

しかし、単純な用途ではこれでいいのだが、複雑な条件のテストでは `wrapper` 関数にオプションを渡して、Redux store の初期設定をしたくなる。これの実現方法は、以下のように少し手を加えればできる[^1]。

```javascript
const wrapper = ({ children }, options = {}) => {
  const mockStore = configureStore();
  const store = mockStore(
    _.merge(
      {
        session: {},
        books: {},
      },
      options.store
    )
  );
  ...
};

const customRender = (ui, options = {}) =>
  render(ui, { wrapper: props => wrapper(props, options.wrapperOptions), ...options });
```

ここでは日和って [lodash の merge()](https://lodash.com/docs/4.17.15#merge) を使った。使う側ではこう。

```javascript
render(<MyComponent {...props} />, {
  wrapperOptions: {
    store: {
      books: { books: [book1, book2] }
    }
  }
});
```



[^1]: [render options: ability to pass props to wrapper · Issue #780 · testing-library/react-testing-library](https://github.com/testing-library/react-testing-library/issues/780)

