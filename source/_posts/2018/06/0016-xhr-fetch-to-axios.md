---
title: '[JavaScript] Fetchの代わりにAxiosを使ってみる'
date: 2018-06-19 11:47:06
tags:
- axios
- node-fetch
---

単純に HTTP で通信したいと思った時、モダンな API だと思っている[Fetch API](https://developer.mozilla.org/ja/docs/Web/API/Fetch_API)を使っていたが
electron-vue のテンプレートを使った時に axios がデフォルトで使われていたので、axios の使い方を覚えてみる。

<!-- more -->

## XHR ライブラリ比較

まず JavaScript における XHR ライブラリはやたらと多い。何があってどんな違いがあるのか全くわからない。

と思っていたら機能比較の一覧を見つけた。これは便利

<a href="https://www.javascriptstuff.com/ajax-libraries/" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

ただ Concise Syntax と Formal Specification が具体的に何を指しているのか良く分からなかった。簡素な記述ができるか？公式の仕様があるか？って意味でいいんだろうか。

重要視する点は Promise サポートだ、正直もう非同期コールバック地獄は勘弁して欲しい。そうすると`fetch系列`か`axios`か`reqwest`の選択肢となる。reqwest は日本語情報がほとんどなかったし、半面 axios は情報が多そうなので、axios という選択肢はそんなに悪くなさそうだ。

## 使ってみる

こんな感じで使うことができた

```javascript
import axiosbase from 'axios'

const axios = axiosbase.create({
  baseURL: 'http://<domain>',
  headers: {
    'Content-Type': 'application/json',
  },
  responseType: 'json',
})

export default function() {
  axios
    .get(`/<address>`)
    .then(res => res.data)
    .catch(error => error)
}
```

node-fetch と使用感を比べてみる、サンプルは以下のコードだ

```js
const fetch = require('node-fetch')

const url = '<url>'
const headers = {}

fetch(url, { headers })
  .then(res =>
    res.json().then(data => ({
      ok: res.ok,
      data,
    }))
  )
  .then(res => {
    if (!res.ok) {
      throw Error(res.data.message)
    } else {
      console.log(res.data)
    }
  })
```

正常にレスポンスが返ってきたらデータをコンソールに表示し、エラー（404 や 500）なら例外をスローするという形だ。やりたいことはシンプルなのに必要以上にごちゃごちゃしている。

原因は FetchAPI の仕様で、404 や 500 を`catch`ではなく`then`で返すところだ。なので 200 との区別を自力（`response.ok`）で判断しなければならない。

それだけならまだしも、response を json に変換するのに`then`を一回噛まさないといけない。その為、2 回目の`then`で OK/NG 判定する為に 1 回目の`then`で json 変換した際`response.ok`を無理やり渡す感じにしている。（1 回目の`then`で`throw`してしまうと、`message`が取得できない）

axios だとその辺がスッキリしているというか、本来こうあるべきだよねというシンプルな実装にできている。400 番台も 500 番台も`catch`側に飛んでくるし、これはとても分かりやすい。

---

余談だけど、node-fetch にも`catch`はある。あるけどどうやらネットワークエラー関係しか`catch`しないみたいだ。400 番台も 500 番台も正常にレスポンスが返ってきている以上正常系という扱いなんだろうか。

## CORS 対応

axios の使い方ではないけど、electron-vue で axios を使って API サーバにアクセスしようとしたら表題の問題に直面した。

electron 側は localhost という扱いの為、クロスドメインに引っかかったみたいだ。今回サーバーは sinatra を使っていたのでそちら側で`Access-Control-Allow-Origin`を許可して対応。

詳しくは[ここ](http://takapi86.hatenablog.com/entry/2017/09/18/172846)を参考にやりました。すごく詳細に書いてあって助かった！

## さいごに

上記の node-fetch の例はさすがに酷いというか、もうちょっとイケてる書き方があるんじゃないかと思うんだけど、それを考える前に axios でいいんじゃないの？という気がしてきた。

実際のところ、400 番台と 500 番台をどう扱うかでどちらを使うか選定する感じがいいのかな。

---

## 追記

async/await にしたら fetch でもスッキリ書けた。

```js
const fetch = require('node-fetch')

const url = '<url>'
const headers = {}

const res = await fetch(url, { headers });
const json = await res.json();
if (!res.ok) {
  throw Error(json.message)
} else {
  console.log(json.data)
}
```
