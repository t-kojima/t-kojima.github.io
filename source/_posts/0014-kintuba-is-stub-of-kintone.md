---
title: '[kintone] kintoneで自動テスト'
date: 2018-06-12 15:43:18
tags:
- kintone
- nodejs
- karma
---

kintone 用の JavaScript ファイルを jasmine や mocha でそのままテストすると、当然ながら kintone オブジェクトはエラーになる。

```js
ReferenceError: kintone is not defined
```

これがちょっとイヤだったので、テストコード内であたかも kintone のように振る舞うオブジェクトを作成した。

<!-- more -->

## kintuba

kintuba という npm モジュールとして作成した。

[https://github.com/t-kojima/kintuba](https://github.com/t-kojima/kintuba)

当初`kintone stub`を省略した`kintub`という名前にしようかと思ったが、ググったらやたらと「きんつば」が出てきたので`kintuba`にした。

一応単体でも動作するけど、ブラウザテストが本筋な気がしたので karma プラグインも一緒に作った。

[https://github.com/t-kojima/karma-kintuba](https://github.com/t-kojima/karma-kintuba)

## 使用例

ここでも kintone のチュートリアル[第 2 回 レコード一覧画面にボタンを置いてみよう！](https://developer.cybozu.io/hc/ja/articles/201767270)のサンプルを使って、テストをしてみたいと思う。

### 準備

テストは karma と mocha、chai を使って行うので、必要なモジュールをインストールする。jasmine でもいいけど、async/await なテストは mocha のほうが書きやすいので mocha 採用で

```bash
yarn add --dev kintuba karma-kintuba
yarn add --dev mocha chai
yarn add --dev karma karma-mocha karma-chai karma-cli karma-chrome-launcher
```

今回はやらないけど、alert とか confirm のテストもするなら sinon があるといい

```bash
yarn add --dev sinon karma-sinon
```

karma.conf.js を作成し、以下のようにする

`frameworks`に kintuba を追記する以外は普通の設定

```js
// Karma configuration
// Generated on Tue May 22 2018 11:49:46 GMT+0900 (東京 (標準時))

module.exports = config => {
  config.set({
    basePath: '',
    frameworks: ['mocha', 'chai', 'kintuba'],
    files: ['src/**/*.js', 'test/**/*.js'],
    exclude: [],
    preprocessors: {},
    reporters: ['mocha'],
    port: 9876,
    colors: true,
    logLevel: config.LOG_INFO,
    autoWatch: true,
    browsers: ['Chrome'],
    singleRun: true,
    concurrency: Infinity
  })
}
```

### テスト対象コード

- src/target.js

```js
;(function() {
  kintone.events.on('app.record.index.show', function(event) {
    // 増殖バグを防ぐ
    if (document.getElementById('my_index_button') !== null) {
      return
    }

    var myIndexButton = document.createElement('button')
    myIndexButton.id = 'my_index_button'
    myIndexButton.innerHTML = '一覧のボタン'
    kintone.app.getHeaderMenuSpaceElement().appendChild(myIndexButton)
  })
})()
```

### テストコード

- test/test.js

```js
/* eslint-disable no-undef */

describe('example', () => {
  it('ボタンが追加されていること', async () => {
    await kintone.events.do('app.record.index.show')
    const button = document.getElementById('my_index_button')
    chai.assert.equal(button.innerHTML, '一覧のボタン')
  })
})
```

これで`karma start`すると、chrome が立ち上がりテストが正常に通る。

特筆すべきところは`do`メソッドのみ

```js
await kintone.events.do('app.record.index.show')
```

kintone では画面の移動などでイベントが発火するが、ローカルテストではその動きはできないので、テストコード側で手動発火させる。それが`do`メソッドだ。

非同期で動作し、`kintone.events.on`で登録したイベントハンドラの戻り値を`kintone.Promise`オブジェクトとして返す。

## 挫折したところ

`app.record.index.show`とか一覧表示のところで、`event.records`の配列をソートしたりフィルタする動作は軒並み再現を諦めた。

kintone から取得できる view のデータでは、ソートやフィルタの`condition`は SQL ライクに記述してあるので、JavaScript の Array を操作できる形に書き換えるのはとても骨だ。逆にテストデータを json ではなく sqlite なんかで管理するようにすればそのままソートやフィルタできたのかもしれないが、それはそれでインパクトでかい、どちらにしても変更量が半端ない感じがしたので TODO 状態で放置してある。

## 作ってみて

個人的には JavaScript の習熟と kintone API への理解が深まった点が大きな収穫だ。ただ`globals.kintone`を無理やり上書きしてたり、`eval`使ってたり、`eslint-disable`をあらゆる所で追記してたりで、本来であればもうちょっとスマートに書けるんじゃないかな？と思うところがかなりある。`kintone.api`関連はまだ未実装なメソッドが多く残ってたりするので、実装を進めると同時にリファクタリングもしていきたい。
