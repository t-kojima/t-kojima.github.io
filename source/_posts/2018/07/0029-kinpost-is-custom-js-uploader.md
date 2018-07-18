---
title: '[Kintone] カスタムJS（CSS)ファイルのアップロードモジュールを作った'
date: 2018-07-19 00:07:37
tags:
- kintone
---

Kintone のカスタム JavaScript ファイルを開発する際、Dropbox の共有リンクを使うと反映が簡単になるという [Tips](https://developer.cybozu.io/hc/ja/articles/201308690-JavaScript%E3%82%AB%E3%82%B9%E3%82%BF%E3%83%9E%E3%82%A4%E3%82%BA%E3%81%AE%E3%83%87%E3%83%90%E3%83%83%E3%82%B0%E3%82%92%E3%81%8B%E3%82%93%E3%81%9F%E3%82%93%E3%81%AB%E3%81%99%E3%82%8B%E3%82%A6%E3%83%A9%E3%83%AF%E3%82%B6) がある。
しかしいくつか問題点（？）があるので、ファイルのアップロードモジュールを作成した。

<a href="https://github.com/t-kojima/kinpost" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

<!-- more -->

# Dropbox 共有リンクの問題点

正直クリティカルではない問題点ばかりだが、いくつかある。

- 共有リンクはインターネット上に公開されてしまう。
- Dropbox サービスに依存している。サービスダウン等で使用不可能になる。
- アップロードするファイル毎にリンクを発行する必要があるので、ファイルが段階的に増えると都度のリンク発行が面倒。

# kinpost

「外部サービスを経由するのが嫌なら、直接アップロードしちゃえばいいじゃない」の精神で、カスタム JavaScript ファイルを Kintone アプリに直接アップロードできるようにしたのが kinpost だ。

kinpost を使うと Dropbox を介さないので、上記共有リンク問題点を気にしなくても良くなる。

…というのは後付けで、本当は「スペースとスペース外でアプリをデータごとコピーする」というモジュールを作っていたのだけど、その中のカスタム JavaScript/CSS ファイルをコピーする箇所が単一のモジュールとして成り立ちそうだったから作ってみた。

当初はコマンドでも実行できるようにしようとしていたが、どうしてもファイルの指定が複雑になりそうだったので取り止めた。

# 使い方

kinpost モジュールは以下のような形で利用できる。

```bash
yarn add --dev kinpost
```

```js
// upload.js

const kinpost = require('kinpost')

kinpost({
  domain: 'example.cybozu.com',
  app: 1,
  username: 'user@example.com',
  password: '**********',
  files: [
    { path: './js/sample.js' },
    { path: './js/mobile.js', platform: 'mobile' },
    { path: './css/global.css', type: 'css' }
  ]
})
```

このファイルをそのまま実行すると、3 つのファイルがアップロードされる。

- PC 用 JavaScript ファイルに `sample.js`
- スマートフォン用 JavaScript ファイルに `mobile.js`
- PC 用 CSS ファイルに `global.css`

素の実行と手間はほとんど変わらないけど、npm スクリプトとして登録しておいても良い。

```js
  "scripts": {
    "upload": "node upload.js"
  }
```

## 処理内容

基本的に Kintone REST API の組み合わせで実装している。

1.  JavaScript/CSS カスタマイズ設定の取得
2.  アップロードするファイルを読み込み
3.  ファイルをアップロードして、ファイルキーを取得
4.  読み込んだカスタマイズ設定の書き換え
5.  JavaScript/CSS カスタマイズ設定の変更
6.  アプリの設定の運用環境への反映

# テスト環境と本番環境

ファイルのアップロードをテスト環境に留めるか、それとも本番環境まで反映させてしまうかは悩んだ（というか今でも迷っている）が、結局本番環境まで反映させている。

テスト環境に留めたほうが良いと思った点は

- deploy の API を叩いてしまうと、他にもテスト環境で止めてある変更点があった場合、それらも巻き添えで公開してしまう。

本番環境まで反映させたほうが良いと思った点は

- 一見ファイルがアップロードされている（Kintone の設定画面を見てもファイルはあるように見える）けど、実際のアプリに反映されてい状態になるので、紛らわしい。

余談だけど、テスト環境にアップロードして deploy API を叩かないでおくと、Kintone の画面では以下のような状態（設定を変更して、「アプリを更新」ボタンを押す直前）になる。

![](/images/29-01.png)

# 考えていること

## 本番環境反映について

本番デプロイの強制はやはりまずい気がしてきた。

本番デプロイはオプションにして、基本はテスト環境への反映にしたほうがいいかもしれない…

### 追記

v1.0.1 で本番デプロイをオプションにした。

## ファイル指定方法

kinpost のファイルの指定方法を見てたらこれはこれで面倒くさい気がしてきた。ファイル指定じゃなくて、ディレクトリ指定もできるようにすると楽かも

こんな感じで

```js
kinpost({
  domain: 'example.cybozu.com',
  app: 1,
  username: 'user@example.com',
  password: '**********',
  files: [],
  dirs: [{ path: './js' }] // jsディレクトリ配下の全ファイルをdesktop/jsとしてアップロード
})
```
