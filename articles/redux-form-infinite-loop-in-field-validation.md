---
title: "Redux Form の Field-Level Validation で無限ループ"
emoji: "➰"
type: "tech"
topics: ["javascript", "redux", "reduxform"]
published: true
---

もう随分と前から報告されている問題[^1] なので今更だとは思うが、半年に一回は嵌っている落とし穴なので、ここに書き残しておきたい。

:::message

検証に使った Redux Form のバージョンは 8.2.1

:::

[Redux Form](https://redux-form.com/) では、以下のように、Field コンポーネントの validate 属性に関数を指定することで、フィールド単位で Validation を指定できる。これを [Field-Level Validation](https://redux-form.com/8.3.0/examples/fieldlevelvalidation/) という。validate 属性には配列を指定できるので、複数の関数を組み合わせることが可能だ。

```javascript
const FieldLevelValidationForm = props => {
  const { handleSubmit, pristine, reset, submitting } = props;
  return (
    <form onSubmit={handleSubmit}>
      <Field
        name="username"
        type="text"
        validate={[required, maxLength15]}
      />
    </form>
  );
}
```

validate 属性に指定している関数のうち、たとえば、`maxLength15` は以下のように実装されている。

```javascript
const maxLength = max => value =>
  value && value.length > max ? `Must be ${max} characters or less` : undefined;
const maxLength15 = maxLength(15);
```

ところで、`maxLength` 関数は他のフォームでも利用できそうなので、別のモジュールに移動して再利用したくなるだろう。

```Javascript
import { maxLength } from 'util/validator/LengthValidator';

const maxLength15 = maxLength(15);
```

そして、ここまで来ると、`maxLength15` 変数が冗長に思えてくる。簡潔にこう書きたい誘惑に駆られる。

```javascript
import { maxLength } from 'util/validator/LengthValidator';
...
      <Field
        name="username"
        type="text"
        validate={[required, maxLength(15)]}
      />
```

もちろん、Render のたびに関数が再作成されるコストはあるものの、たいていの場合、大きな問題にはならない[^2]。

だが、気が効いているように思えるこのちょっとした変更は、無情にもあなたのフォームを壊してしまう。

```
Violation: Maximum update depth exceeded. This can happen when a component repeatedly calls setState inside componentWillUpdate or componentDidUpdate. React limits the number of nested updates to prevent infinite loops.
    at invariant (webpack:///./node_modules/react-dom/cjs/react-dom.development.js?:55:15)
    at scheduleWork (webpack:///./node_modules/react-dom/cjs/react-dom.development.js?:19916:5)
...
```

ログを確認してみると、`@@redux-form/UNREGISTER_FIELD` と `@@redux-form/REGISTER_FIELD` のふたつのアクションがひたすら繰り返されている。**無限ループだ。**

実は、この問題については、[公式の API リファレンスにも注意書きがある](https://redux-form.com/8.2.2/docs/api/field.md/#-code-validate-array-lt-function-gt-value-allvalues-props-name-gt-error-code-optional-)。

> Note: if the validate prop changes the field will be re-registered.

というわけで、validate 属性に渡す関数はコンポーネントの外側で定義したものを使うのが一番いいだろう。

## validate 属性に渡す関数で props を使いたい

では、validate 属性に渡す関数で、コンポーネントに渡された props を参照したい場合はどうすればいいだろうか。これは簡単で、validate 属性に渡す関数にはフィールドの値 `value` の他にも以下の引数が渡される。

- `value` - フィールドの値
- `allValues` - フォームのすべてのフィールドの値
- `props` - props
- `name` - フィールド名

これで、props を参照する関数も実装できる。

```javascript
const validateUniqueBook = (value, _values, props) => {
  const { books } = props;
  return books.every(b => b.isbn !== value) ? undefined : "You already have this book.";
});
```

[^1]: [Field level validation & warning · Issue #4017 · redux-form/redux-form](https://github.com/redux-form/redux-form/issues/4017)
[^2]: [Passing Functions to Components – React](https://reactjs.org/docs/faq-functions.html#how-do-i-bind-a-function-to-a-component-instance)

