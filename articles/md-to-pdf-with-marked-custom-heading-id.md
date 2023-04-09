---
title: "md-to-pdf で Markdown から PDF に変換: 独自の Heading ID を指定する"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["markdown"]
published: false
---

業務では、社内で扱う情報や外部へ提出する文書など、PDF 形式で情報を共有する必要に迫られることがある。しかし、エンジニアにとっては、プログラムのソースコードやドキュメントなどは普段から Markdown や plain text で取り扱い、そのまま共有することに慣れているため、PDF 形式での情報共有に苦手意識を抱く人も多いのではないだろうか。

一方で、世の中を見渡してみると、非エンジニアな人々には Markdown アレルギーな体質の人が多いらしい。ここは譲り合いの精神で、Markdown を PDF に変換してあげるべきだろう（彼らだって、本当は Word ドキュメントでほしいのを我慢しているのかもしれないし...）。

## md-to-pdf の紹介

VSCode エディタには、Markdown ファイルを PDF にエクスポートするための多くの拡張機能が存在する。例えば、[Markdown Preview Enhanced](https://marketplace.visualstudio.com/items?itemName=shd101wyy.markdown-preview-enhanced) や [Markdown PDF](https://marketplace.visualstudio.com/items?itemName=yzane.markdown-pdf) を使用することで、ひとつのMarkdown ファイルを簡単に PDF に変換することができる。

しかし、今回は別の手段として [md-to-pdf](https://github.com/simonhaenisch/md-to-pdf) [^1] を紹介したい。md-to-pdfを使用することで、複数のMarkdown ファイルをひとつの PDF 文書にまとめることができる。それだけでなく、コマンドラインで操作できるため、CI/CDパイプラインに組み込むことができる（今後もアップデートがありうる文書では重要）。

また、`npm` で簡単にインストールできるし、`md-to-pdf` のコマンド一発で PDF を出力できるので、普段使いのツールとしても気軽に使うことができる。

md-to-pdf の基本的な使用方法に関しては[公式のページ](https://www.npmjs.com/package/md-to-pdf)や他の記事に譲るとして、この記事では、Markdown の Heading に独自のIDを割り当てる方法を紹介したい。

## md-to-pdf で独自の Heading ID を指定したい

md-to-pdf のデフォルト設定では、Heading の ID がタイトルから自動で振られる。

しかし、これではタイトルを変更すると ID も変わってしまい、他の箇所から参照しているリンクが壊れてしまう。やってみると分かるのだが、マニュアルのように目次が必須の文書だと、この仕様はけっこうつらい。

そこで、この記事では、md-to-pdf を使って独自の ID を指定する方法を紹介したい。これによって、Heading に必要な ID を指定することができ、タイトルの変更にも柔軟に対応できるようになる。

## Markdown の拡張構文: 独自 Heading ID

まずは、Markdown の[拡張構文](https://www.markdownguide.org/extended-syntax)で[独自の Heading ID を指定する構文](https://www.markdownguide.org/extended-syntax/#heading-ids)を確認しよう。

```
### My Great Heading {#custom-id}
```

この Markdown は、次の HTML に変換される。

```html
<h3 id="custom-id">My Great Heading</h3>
```

## md-to-pdf での設定方法

md-to-pdf が Markdown の変換に利用している [Marked](https://github.com/markedjs/marked) では、デフォルトでは自動的に ID が割り振られるだけだが、[拡張機能](https://marked.js.org/using_advanced#extensions)をインストールすることでさまざまな拡張が可能になる。ここでは、[marked-custom-heading-id](https://www.npmjs.com/package/marked-custom-heading-id) 拡張機能を使って、独自の ID 指定を可能にする。

まず、`marked-custom-heading-id`をインストールしよう。

```
$ npm i -D marked-custom-heading-id
```

次に、md-to-pdfの設定ファイル（`md-to-pdf.config.js`）に、Marked のオプションと拡張機能を追加する。

```js
// md-to-pdf configuration
const customHeadingId = require("marked-custom-heading-id");

module.exports = {
  marked_options: {
    headerIds: false,
    smartypants: true,
  },
  marked_extensions: [customHeadingId({})],
  pdf_options: {
    format: "A5",
    margin: "30mm 20mm",
    printBackground: true,
  },
};
```

このように設定ファイルを作成し、`md-to-pdf`コマンドを実行する際に、`--config-file`オプションを指定して、設定ファイルを渡すことができる。

```
$ md-to-pdf --config-file md-to-pdf.config.js
```

これで、md-to-pdf で独自の Heading ID を使えるようになった。

md-to-pdf には他にもタイトルページの設定やカスタムスタイルシートによる調整など、痒いところに手が届く設定が用意されている。オプションについては [README](https://github.com/simonhaenisch/md-to-pdf/blob/master/readme.md) に書かれているので、使う前に一度確認してみてほしい。

また、Markdown には多くの[拡張構文](https://www.markdownguide.org/extended-syntax)があり、Markdown プロセッサによってはオプションでサポートされているものも多い。たとえば、今回紹介した Marked では[拡張機能](https://marked.js.org/using_advanced#extensions)によって構文を拡張することができる。何かドキュメントを書くときに Markdown の表現力では物足りなくなってきたらこれらの拡張に目を通してみるのもいいだろう。

[^1]: md-to-pdfは、シンプルなCLIツールであり、[Puppeteer](https://github.com/puppeteer/puppeteer) を使って Chrome/Chromium によってレンダリングされた HTML を PDF として出力する。そのため、Web コンテンツとの相性も良く、PDF 変換に愛用している。コードベースも小さいので、理解が容易なのも魅力だ。
