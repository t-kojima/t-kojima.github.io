---
title: 'Browserifyで"fs.readFileSync is not a function"'
date: 2018-07-04 10:28:41
tags:
- browserify
- nodejs
---

以前作成した [kintuba](https://github.com/t-kojima/kintuba)という npm モジュールで、モジュールバンドラに Browserify を使っていた。このモジュールのテストでは、本来 Browserify でバンドルしたファイルに対してテストすべきなんだけど、ずっとバンドル前のファイルでテストしてしまっていた。

今回そこを修正すべくバンドル後のファイルでテストが流れるよう修正してみた。

<!-- more -->

## 突然のエラー

バンドル前のファイルでは問題なくテストがパスするんだけど、バンドル後のファイルだとエラーになってしまった…！

```bash
TypeError: fs.readFileSync is not a function
```

調べてみると有名なエラーっぽくて、Browserify が定義する `require` と node の `require` が混ざってしまってえらいこっちゃな感じで、fs がロードできない問題らしい。

<a href="https://stackoverflow.com/questions/16640177/browserify-with-requirefs" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

## brfs

解決策その 1 brfs というモジュールを追加で使う。

<a href="https://github.com/browserify/brfs" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

example にある通り、これが

```js
var html = fs.readFileSync(__dirname + '/robot.html', 'utf8')
```

こうなる

```js
var html = '<b>beep boop</b>\n'
```

…と書いてあるんだけど、実際やってみても変換されなかった。なんでだ？

## browserify-fs

解決策その 2 browserify-fs というモジュールを追加で使う。

<a href="https://github.com/mafintosh/browserify-fs" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

~~これも使ってみたが、何故か上手く動かない。こっちは brfs 以上に情報が少ないので断念~~

`level-filesystem` を使っている場合に有効らしい。もちろんそんなのは使っていない。

## webpack

解決策その 3 モジュールバンドラを webpack に変える。

<a href="https://webpack.js.org/" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

最終手段（？）として、Browserify やめて Webpack にしちゃおう

コンフィグを用意して…

```js
// webpack.config.js
const path = require('path')

module.exports = {
  mode: 'development',
  target: 'node',
  entry: './lib/index.js',
  output: {
    filename: 'index.js',
    path: path.join(__dirname, '.'),
  },
}
```

ビルドするだけ！

```bash
yarn webpack
```

うまくいきました。

## さいごに

テストはバンドル後のファイルでやること。身に沁みました。

---

## 7/6 追記

**全然解決してませんでした。**

### そもそもの勘違い

browserify や webpack が色々よしなにやってくれているので勘違いしていたけど、**web で fs モジュールはロードできない**

今回 kintuba では以下 2 点を同一のスクリプトで動かそうとしていた

- node で kintuba を動かす場合は fs モジュールでローカルファイルをロードする
- karma で kintuba を動かす場合はアップロードされたファイルを adapter が fetch でロードする

結論から言うと fs を組み込めば karma でエラーになり、fs を組み込まなければ node でファイルロードができない。これは webpack しようが babel しようが polyfill しようが変わらない当たり前すぎる帰結。

### Broserify では

Browserify では fs が[デフォルトで ignore](https://qiita.com/terrierscript/items/1fe19d32011e39ed8e2b#%E3%81%9D%E3%81%AE6-ignore-i%E3%82%AA%E3%83%97%E3%82%B7%E3%83%A7%E3%83%B3)になっていたので karma では問題なく動く、そして Browserify 前のファイルでテストは通っていたので問題無いと思い込んでいた。

### ではどうするの？

fs を利用するモジュールを分離し、node からしか呼ばないようにする。これでファイルロードに関する部分 fixture・schema モジュールとして kintuba の core から外出しできた。

これで node の場合は fixture・schema モジュールがファイルをロードし、karma の場合は karma の adapter がファイルをロードする。kintuba の core はそれらを受け取る API のみを提供するので、関心毎の分離が以前より明確にできた。
