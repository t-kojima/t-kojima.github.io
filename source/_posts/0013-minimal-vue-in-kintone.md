---
title: '[kintone] 最小構成のvue.jsを使ってみる'
date: 2018-06-12 09:54:24
tags:
- vue
- kintone
---

最小構成という言葉が好きです。
新たなフレームワークやモジュールを導入する時、モダンな構成や流行りの構成を試す前に最小構成を考えてみると理解が捗る気がする。

<!-- more -->

## kintone チュートリアルを vue.js でやる

kintone のチュートリアル[第 2 回 レコード一覧画面にボタンを置いてみよう！](https://developer.cybozu.io/hc/ja/articles/201767270)では単純なボタンを追加するサンプルコードがある。

これを vue.js でやってみよう。

### vue.js を読み込ませる

アプリの設定-JavaScript/CSS カスタマイズで`https://cdn.jsdelivr.net/npm/vue/dist/vue.js`を追加する。これでカスタム JavaScript 内で`Vue`が使えるようになる。

### カスタム JavaScript を作成する

サンプルコードに Vue オブジェクトを追加する。

```js
;(function() {
  kintone.events.on('app.record.index.show', function(event) {
    var myIndexButton = document.createElement('button')
    myIndexButton.id = 'my_index_button'
    myIndexButton.innerHTML = '{{ message }}'
    kintone.app.getHeaderMenuSpaceElement().appendChild(myIndexButton)

    var app = new Vue({
      el: '#my_index_button',
      data: {
        message: 'Hello Vue!'
      }
    })
  })
})()
```

[参考 : https://developer.cybozu.io/hc/ja/articles/201767270](https://developer.cybozu.io/hc/ja/articles/201767270)

ポイントは以下 3 点

- `myIndexButton.innerHTML`を`{{ message }}`にし、Vue の message が表示されるようにする。
- Vue の el は`#my_index_button`とし、作成したボタンにバインドされるようにする。
- Vue は`kintone.app.getHeaderMenuSpaceElement().appendChild`よりも **後** に定義する。（前に定義するとバインドされない）

これで「Hello Vue!」というボタンが追加される。

### Element を使う

[Element](http://element.eleme.io/#/en-US)も使ってみる。[Steps コンポーネント](http://element.eleme.io/#/en-US/component/steps)を余白に追加してみよう。

vue.js と同様に、JavaScript/CSS カスタマイズで`https://unpkg.com/element-ui/lib/index.js`と`https://unpkg.com/element-ui/lib/theme-chalk/index.css`を追加する。

コードは以下のものになる。

```js
;(function() {
  console.log(kintone)

  kintone.events.on('app.record.index.show', function(event) {
    const button = document.createElement('button')
    button.id = 'button'
    button.innerHTML = 'count up'
    button.addEventListener('click', () => {
      app.step += 1
    })
    kintone.app.getHeaderMenuSpaceElement().appendChild(button)

    const div = document.createElement('div')
    div.id = 'app'
    div.innerHTML = `
        <el-steps :active="step" align-center>
            <el-step title="Step 1" description="Some description"></el-step>
            <el-step title="Step 2" description="Some description"></el-step>
            <el-step title="Step 3" description="Some description"></el-step>
            <el-step title="Step 4" description="Some description"></el-step>
        </el-steps>
        `
    kintone.app.getHeaderSpaceElement().appendChild(div)

    var app = new Vue({
      el: '#app',
      data: {
        step: 1
      }
    })
  })
})()
```

すると余白に`Stepsコンポーネント`が表示された。

![](/images/13-01.png)

また、ちょっと動きも入れている。

`count up`というボタンを追加し、クリックする度にステップが動くのだ。

Element を使うと簡単に見栄えが良くなるのでいいね

## 実行環境

- Vue.js 2.5.16
- Element 2.4.1
