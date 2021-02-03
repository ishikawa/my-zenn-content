---
title: "Jest-Expo でも Random と Crypto が動いてほしい"
emoji: "🙏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [expo, jest]
published: true

---

::: message

（2021/02/04 追記）この記事で紹介したモックをそれぞれ [node-expo-crypto](https://www.npmjs.com/package/node-expo-crypto) と [node-expo-random](https://www.npmjs.com/package/node-expo-random) として公開しました。

:::

Expo プロジェクトを Jest でテストする方法については [Expo の TypeScript プロジェクトで自動テスト](https://zenn.dev/takanori_is/articles/setup-jest-for-expo-typescript-project)で紹介した。しかし、[jest-expo](https://www.npmjs.com/package/jest-expo) を使うと **Expo SDK の提供する API はダミーの値や `undefined` を返すようになってしまう。**[^1]

まあ、Expo SDK の API はターゲット環境となる iOS/Android/Web のネイティブ API を呼び出しているわけで、Jest が実行される Node 環境でこれらが動かないのだから当然だ。

## それでも動いてほしいときはある

今回、テストしたかったのは以下のような関数。

```typescript
import * as Random from 'expo-random';
import * as Crypto from 'expo-crypto';

/**
 * Returns a promise that generates cryptographic random hash digest.
 */
export function randomSHA256Async(): Promise<string> {
  const bytes = Random.getRandomBytes(8);
  const value = bytes.reduce((accumulator, value) => accumulator + String.fromCharCode(value), '');

  return Crypto.digestStringAsync(Crypto.CryptoDigestAlgorithm.SHA256, value);
}
```

Expo の [Random](https://docs.expo.io/versions/latest/sdk/random/) モジュールと [Crypto](https://docs.expo.io/versions/latest/sdk/crypto/) モジュールを使って、ランダムな文字列を生成する関数である。[^2]

単純に Expo SDK を呼び出すだけならテストする必要はないのだが、バイト列 (`Uint8Array`) を文字列に変換する処理が入っているので、やはりテストはしておきたい。しかし、上記のとおり、`Random.getRandomBytes()` が `undefined` を返すので、このままではテストができない。

```typescript
$ npm run test ./test/utils/Random.test.ts

 FAIL  test/utils/Random.test.ts
  randomSHA256Async
    ✕ generate one (4 ms)

  ● randomSHA256Async › generate one

    TypeError: Cannot read property 'length' of undefined

       6 |  */
       7 | export function randomSHA256Async(): Promise<string> {
    >  8 |   const bytes = Random.getRandomBytes(8);
```

[こちらの Issue](https://github.com/expo/expo/issues/6980#issuecomment-606863968) では、それぞれのモジュールをモックする方法が勧められている。というわけで、`jest.mock()` を使ってテストを書いてみよう。

```typescript
import { randomSHA256Async } from 'utils/Random';

jest.mock('expo-random/build/ExpoRandom', () => ({
  getRandomBytes: jest.fn(() => new Uint8Array([1, 2, 3, 4, 5, 6, 7, 8]))
}));
jest.mock('expo-crypto/build/ExpoCrypto', () => ({
  digestStringAsync: jest.fn(async () => 'e46bbb550892006aa4dfbc5813f1e0625b9ac4a76dab5c6632d154117a8707f7')
}));

describe('randomSHA256Async', () => {
  test('generate one', async () => {
    const digest = await randomSHA256Async();

    expect(digest).toHaveLength(64);
    expect(digest).toMatch(/^[0-9a-f]+$/);
  });
});
```

これでテストが通る。

```bash
$ npm run test ./test/utils/Random.test.ts

 PASS  test/utils/Random.test.ts
  randomSHA256Async
    ✓ generate one (3 ms)
```

やったー！😆

## やっぱり、ちゃんと動いてほしい

テストが通って喜んだのも束の間「これってテストになってるのかな？🤔」と不安に襲われる。

というか Node 環境には乱数もハッシュ関数も [crypto モジュール](https://nodejs.org/api/crypto.html)で用意されているので、それらで実装すればよさそうだ。

```typescript
import crypto from 'crypto';

jest.mock('expo-random/build/ExpoRandom', () => ({
  getRandomBytes: jest.fn((byteCount: number) => {
    const buffer = crypto.randomBytes(byteCount);
    return new Uint8Array(buffer.buffer);
  })
}));
...
```

しかし、これは以下のエラーが出て動かない。

```bash
ReferenceError: /path/to/test/utils/Random.test.ts: The module factory of `jest.mock()` is not allowed to reference any out-of-scope variables.
Invalid variable access: crypto
Allowed objects: Array, ArrayBuffer, Atomics, BigInt, BigInt64Array, BigUint64Array, Boolean, Buffer, DataView, Date, Error, EvalError, Float32Array, Float64Array, Function, GLOBAL, Generator, GeneratorFunction, ...
Note: This is a precaution to guard against uninitialized mock variables. If it is ensured that the mock is required lazily, variable names prefixed with `mock` (case insensitive) are permitted.
```

エラーで書かれている通り、`jest.mock()` では、外側のスコープにあるオブジェクトは一部以外はアクセスできない。このように変えてやる必要がある。

```typescript
jest.mock('expo-random/build/ExpoRandom', () => ({
  getRandomBytes: jest.fn((byteCount: number) => {
    const crypto = require('crypto');
    const buffer = crypto.randomBytes(byteCount);
    return new Uint8Array(buffer.buffer);
  })
}));
```

## すべてのテストで動かす

Random にしろ Crypto にしろ、他のテストでも使う可能性があるので、せっかくならグローバルで有効になるようにしよう。まずはモックするファイルを用意する。

```typescript:test/utils/expo-random/index.ts
jest.mock('expo-random/build/ExpoRandom', () => ({
  getRandomBytes: jest.fn((byteCount: number) => {
    const crypto = require('crypto');
    const buffer = crypto.randomBytes(byteCount);
    return new Uint8Array(buffer.buffer);
  }),
  getRandomBytesAsync: jest.fn((byteCount: number) => {
    const crypto = require('crypto');
    const buffer = crypto.randomBytes(byteCount);
    return Promise.resolve(new Uint8Array(buffer.buffer));
  })
}));
```

```typescript:test/utils/expo-crypto/index.ts
jest.mock('expo-crypto/build/ExpoCrypto', () => ({
  digestStringAsync: jest.fn((algorithm: string, data: string, options = { encoding: 'hex' }) => {
    const crypto = require('crypto');
    const hash = crypto.createHash(algorithm.toLowerCase().replace('-', ''));

    hash.update(data);
    return Promise.resolve(hash.digest(options.encoding ?? 'hex'));
  })
}));
```

あとは、これらを Jest の [`setupFilesAfterEnv`](https://jestjs.io/docs/en/configuration#setupfilesafterenv-array) で読み込むようにする。

```typescript:jest.config.ts
const config: Config.InitialOptions = {
  preset: 'jest-expo',
  ...
  setupFilesAfterEnv: [
    './test/utils/expo-crypto/index.ts',
    './test/utils/expo-random/index.ts'
  ]
};
```

これで、すべてのテストで Expo の Random モジュールと Crypto モジュールが期待通りに動くようになる。

[^1]: [src/preset/expoModules.js](https://github.com/expo/expo/blob/ios/2.16.1/packages/jest-expo/src/preset/expoModules.js) で定義された API 一覧を、[こんな感じ](https://github.com/expo/expo/blob/ios/2.16.1/packages/jest-expo/src/preset/setup.js#L41)でモックしているようだ。
[^2]: ちなみに、OpenID Connect の nonce を生成するために使う。