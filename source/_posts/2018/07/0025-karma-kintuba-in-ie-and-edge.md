---
title: [karma-kintuba] ES2017で書かれたモジュール・テストをIEで動かしたい
date: 2018-07-06 11:43:55
tags:
- karma
- webpack
- babel
---

![babel logo](/images/25-babel.png)

[karma-kintuba](https://github.com/t-kojima/karma-kintuba)はモダンブラウザでしか動かない。
~~しかし MS Edge はモダンブラウザのはず（？）なのに動かない。これはトランスパイルで何とかならないかと思い、babel してみた。~~

Edge は動きました。以後 IE との闘いです。

<!-- more -->

## トランスパイルの対象

トランスパイルの対象になるのは以下 3 つだ

- kintuba 本体
- karma-kintuba の adapter
- karma のテストコード

ちなみに karma-kintuba の adapter とは、karma-kintuba が kintuba 本体と一緒にロードするスクリプトで、テスト時にローカルファイルを読み込む為に必要となる。

## karma-kintuba のテスト環境

作った。

<a href="https://github.com/t-kojima/kintuba-test" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

適当にソースとテストコードを用意した。

## 素の状態でテスト

まずは素の状態（というか現時点の最新の状態）で Chrome、Firefox、IE、Edge、PhantomJS に対してテストを流してみる。

| 対象                     | 記法              | 処理    |
| ------------------------ | ----------------- | ------- |
| kintuba 本体             | CommonJS + ES2016 | webpack |
| karma-kintuba の adapter | ES2016            |         |
| karma のテストコード     | ES2016            |         |

### Chrome

問題なくテスト OK

```bash
SUMMARY:
√ 19 tests completed
Done in 8.49s.
```

### Firefox

こちらも OK

```bash
SUMMARY:
√ 19 tests completed
Done in 8.27s.
```

### IE

盛大にエラーが出た、省略したけどこれ以外にも 7 点ほど構文エラーが出ている

```bash
IE 11.0.0 (Windows 10 0.0.0) ERROR
  {
    "message": "構文エラーです。\nat node_modules/karma-kintuba/adapter.js:1:47",
    "str": "構文エラーです。\nat node_modules/karma-kintuba/adapter.js:1:47"
  }

Finished in 0.17 secs / 0 secs @ 12:07:16 GMT+0900 (東京 (標準時))

SUMMARY:
√ 0 tests completed
IE 11.0.0 (Windows 10 0.0.0) ERROR
  {
    "message": "構文エラーです。\nat node_modules/kintuba/index.js:1:962",
    "str": "構文エラーです。\nat node_modules/kintuba/index.js:1:962"
  }

× Error while running the tests! Exit code: 1
```

### Edge

`※事前に「試験的なJavaScript機能を有効にする」必要がある。Edgeを立ち上げてアドレスに about:flag を入力し、設定画面で設定する。`

OK だった。

「試験的な JavaScript 機能を有効にする」に気づかず、ずっと async/await がエラーを吐いていたが、そこさえクリアできれば問題なかった。

```bash
SUMMARY:
√ 19 tests completed
Done in 5.61s.
```

### PhantomJS

こちらもエラー、SyntaxError が 2 件だ

```bash
PhantomJS 2.1.1 (Windows 8 0.0.0) ERROR
  {
    "message": "SyntaxError: Unexpected token '>'\nat node_modules/karma-kintuba/adapter.js:1:0",
    "str": "SyntaxError: Unexpected token '>'\nat node_modules/karma-kintuba/adapter.js:1:0"
  }

Finished in 0.076 secs / 0 secs @ 12:10:06 GMT+0900 (東京 (標準時))

SUMMARY:
√ 0 tests completed
PhantomJS 2.1.1 (Windows 8 0.0.0) ERROR
  {
    "message": "SyntaxError: Unexpected token '>'\nat node_modules/kintuba/index.js:1:0",
    "str": "SyntaxError: Unexpected token '>'\nat node_modules/kintuba/index.js:1:0"
  }

× Error while running the tests! Exit code: 1
```

## IE との闘い

### adapter を webpack(babel+polyfill)

Chrome、Firefox、Edge がクリアできたので正直いいかなって気分だけど、IE で動くまで頑張ってみる。

まず adapter の構文エラー、`adapter.js:1:47` は `=>` を指している。そうだよね、ES2015 の書き方はダメだよねってことで、adapter を webpack する。

| 対象                     | 記法              | 処理          |
| ------------------------ | ----------------- | ------------- |
| kintuba 本体             | CommonJS + ES2016 | webpack       |
| karma-kintuba の adapter | ES2016            | webpack+babel |
| karma のテストコード     | ES2016            |               |

karma-kintuba プロジェクトに webpack と babel をインストール

```bash
yarn add --dev webpack webpack-cli babel-core babel-loader babel-preset-env
```

そして以下の設定で webpack する

```js
// webpack.config.js
const path = require('path')

module.exports = {
  mode: 'development',
  entry: './adapter.js',
  output: {
    filename: 'bundle.adapter.js',
    path: path.join(__dirname, '.'),
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: [
          {
            loader: 'babel-loader',
            options: {
              presets: [['env', { modules: false }]],
            },
          },
        ],
      },
    ],
  },
}
```

できた adapter をテストプロジェクトに移植して…テスト実行！

```bash
IE 11.0.0 (Windows 10 0.0.0) ERROR
  {
    "message": "'regeneratorRuntime' は定義されていません。\nat :6:3\n\nReferenceError: 'regeneratorRuntime' は定義されていません。\n   at Anonymous function (eval code:6:3)\n   at eval code (eval code:5:1)\n
  at ./adapter_e.js (node_modules/karma-kintuba/adapter.js:96:1)\n   at __webpack_require__ (node_modules/karma-kintuba/adapter.js:20:12)\n   at Anonymous function (node_modules/karma-kintuba/adapter.js:84:11)\n   at Global code (node_modules/karma-kintuba/adapter.js:1:11)",
    "str": "'regeneratorRuntime' は定義されていません。\nat :6:3\n\nReferenceError: 'regeneratorRuntime' は定義されていません。\n   at Anonymous function (eval code:6:3)\n   at eval code (eval code:5:1)\n   at ./adapter_e.js (node_modules/karma-kintuba/adapter.js:96:1)\n   at __webpack_require__ (node_modules/karma-kintuba/adapter.js:20:12)\n   at Anonymous function (node_modules/karma-kintuba/adapter.js:84:11)\n
 at Global code (node_modules/karma-kintuba/adapter.js:1:11)"
  }

Finished in 0.105 secs / 0 secs @ 14:06:50 GMT+0900 (東京 (標準時))

SUMMARY:
√ 0 tests completed
```

はい、エラーが変わりました。regeneratorRuntime が無い…と

[Node.js & webpack & babel で「 regeneratorRuntime is not defined」が発生する場合の対処](https://qiita.com/devneko/items/c7ddb31f504c8c2a5ac5)を見るに、blacklist に入れるのがベストとあるが、今回 node ではないのでこれでいいのかわからないのと、そもそも babel6 から blacklist は廃止されてることから、polyfill するしかないかなと思う。

| 対象                     | 記法              | 処理                   |
| ------------------------ | ----------------- | ---------------------- |
| kintuba 本体             | CommonJS + ES2016 | webpack                |
| karma-kintuba の adapter | ES2016            | webpack+babel+polyfill |
| karma のテストコード     | ES2016            |                        |

```bash
yarn add --dev babel-polyfill
```

```js
// webpack.config.js
const path = require('path')

module.exports = {
  mode: 'development',
  entry: ['babel-polyfill', './adapter.js'],
  output: {
      // ... 省略
```

polyfill して再度テストを回すと、adapter 関係のエラーは無くなった。しかしまだエラーはある、kintuba 本体だ。

```bash
IE 11.0.0 (Windows 10 0.0.0) ERROR
  {
    "message": "構文エラーです。\nat node_modules/kintuba/index.js:1:962",
    "str": "構文エラーです。\nat node_modules/kintuba/index.js:1:962"
  }

Finished in 0.222 secs / 0 secs @ 14:31:06 GMT+0900 (東京 (標準時))

SUMMARY:
√ 0 tests completed
```

### kintuba とテストコードでも webpack(babel+polyfill)

エラーの対応は同じで、kintuba 本体側でも babel+polyfill を掛けてあげる

| 対象                     | 記法              | 処理                   |
| ------------------------ | ----------------- | ---------------------- |
| kintuba 本体             | CommonJS + ES2016 | webpack+babel+polyfill |
| karma-kintuba の adapter | ES2016            | webpack+babel+polyfill |
| karma のテストコード     | ES2016            |                        |

```bash
IE 11.0.0 (Windows 10 0.0.0) ERROR
  {
    "message": "構文エラーです。\nat test/add_button_test.js:3:24",
    "str": "構文エラーです。\nat test/add_button_test.js:3:24"
  }

Finished in 0.12 secs / 0 secs @ 14:44:46 GMT+0900 (東京 (標準時))

SUMMARY:
√ 0 tests completed
```

ほい、今度はテストコードでエラー

#### テストコードのトランスパイル

| 対象                     | 記法              | 処理                   |
| ------------------------ | ----------------- | ---------------------- |
| kintuba 本体             | CommonJS + ES2016 | webpack+babel+polyfill |
| karma-kintuba の adapter | ES2016            | webpack+babel+polyfill |
| karma のテストコード     | ES2016            | webpack+babel+polyfill |

テストコードの webpack と polyfill は karma プラグインとして追加する。まずは webpack のみ適用してみる。

```bash
yarn add --dev karma-webpack babel-core babel-loader babel-preset-env
```

karma の設定ファイルに preprocessers と webpack を追加

```js
// karma.conf.js
module.exports = config => {
  config.set({
    basePath: '',
    frameworks: ['mocha', 'chai', 'kintuba'],
    files: [
      'src/**/*.js',
      'test/**/*.js',
      {
        pattern: '.kintuba/**/*.json',
        watched: false,
        included: false,
        served: true,
        nocache: false,
      },
    ],
    exclude: [],
    preprocessors: {
      'test/**/*.js': ['webpack'],
    },
    webpack: {
      mode: 'development',
      module: {
        rules: [
          {
            test: /\.js$/,
            exclude: /node_modules/,
            use: [
              {
                loader: 'babel-loader',
                options: {
                  presets: ['env'],
                },
              },
            ],
          },
        ],
      },
    },
    reporters: ['mocha'],
    port: 9876,
    colors: true,
    logLevel: config.LOG_INFO,
    autoWatch: true,
    // browsers: ['Chrome', 'Firefox', 'PhantomJS', 'Edge', 'IE'],
    browsers: ['IE'],
    singleRun: true,
    concurrency: Infinity,
  })
}
```

これで実行すると…テストは動くようになった

```bash
  example
    √ ボタンが追加されていること
  app.record.index.show
    × "before all" hook
  app.report.show
    × "before each" hook for "イベントが発火すること"
  kintone関数
    × "before all" hook

Finished in 0.02 secs / 0.002 secs @ 14:49:21 GMT+0900 (東京 (標準時))

SUMMARY:
√ 1 test completed
× 3 tests failed
```

#### karma-polyfill で fetch 回避

しかし以下のエラーが出ている

```bash
    × "before all" hook
      IE 11.0.0 (Windows 10 0.0.0)
    ReferenceError: 'fetch' は定義されていません。
```

fetch が無い、polyfill が必要だ

```bash
yarn add --dev karma-polyfill
```

frameworks に polyfill を追加するのと、`polyfill: ['fetch']` を追加する。

```js
// karma.conf.js
module.exports = config => {
  config.set({
    basePath: '',
    frameworks: ['mocha', 'chai', 'kintuba', 'polyfill'],
    polyfill: ['fetch'],
    // ...省略
```

#### mocha のタイムアウト

fetch のエラーは解消されるようになった

```bash
Finished in 14.526 secs / 12.083 secs @ 14:53:00 GMT+0900 (東京 (標準時))

SUMMARY:
√ 8 tests completed
× 6 tests failed
```

が、次の問題、タイムアウトしてしまう。

```bash
Timeout of 2000ms exceeded. For async tests and hooks, ensure "done()" is called; if returning a Promise, ensure it resolves.
```

これはタイムアウトを伸ばす。karma.conf.js に以下を追加する。

```js
    client: {
      mocha: {
        timeout: 10000
      }
    },
```

結果、テストはオールクリアーできた！

```bash
SUMMARY:
√ 19 tests completed
Done in 30.33s.
```

全部のブラウザを有効にして流しても OK だった。

```bash
SUMMARY:
√ 95 tests completed
Done in 36.06s.
```

## イケたかと思いきや

今度は node でのテストがコケるようになってしまった。

```bash
ReferenceError: regeneratorRuntime is not defined
```

kintuba の webpack.config.js で preset のターゲットを node にすると回避はできるが、逆に web 側がまたコケるようになった。

```js
  "presets": [
    [ "env", {
      "targets": {
        "node": "current"
      }
    } ]
  ]
```

ここでまた web と node の環境差で板挟みになってしまった。やはり両方で動くようトランスパイルするのは無謀なんだろうか…

無謀なんだろうな

## さいごに

IE との闘いは勝ったのか負けたのか有耶無耶な結果だったけど、結局 kintuba は babel も polyfill も無しでモダンブラウザのみのサポートとすることにした。

## 実行環境

- Node 8.9.1
- Karma 2.0.4
  - karma-webpack 3.0.0
  - karma-polyfill 1.0.0
- Webpack 4.15.1
- Babel
  - babel-core 6.26.3
  - babel-loader 7.1.5
  - babel-preset-env 1.7.0
  - babel-polyfill 6.26.0
