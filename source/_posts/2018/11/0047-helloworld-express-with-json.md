---
title: '[Node] Expressを使って最速でJSONを返す'
date: 2018-11-02 17:13:38
categories:
  - ['Node', 'Express']
tags:
  - nodejs
  - express
---

皆さん WebAPI で JSON を返したいですよね？Node.js 環境では[json-server](https://github.com/typicode/json-server)とかもあるけれど、ベーシックに Express でやってみたい。

ちなみに最速はレスポンスではなくて作業量的な意味。

<!-- more -->

# Express でハロワ

とりあえずハロワ

yarn で express をインストールする。

```bash
yarn init -y
yarn add express
```

そしたらエントリポイントとなるファイルを作成する。とりあえず`app.js`としとく。

```js
const express = require('express')
const app = express()

app.get('/', (req, res) => res.send('hello world!'))

app.listen(3000, () => console.log('http://localhost:3000'))
```

うーん、4 行！かんたん

最後の`console.log`は無くてもいいが、アドレスを入れておくと起動後に即開けるので楽ちん

```bash
$ node app
http://localhost:3000
```

実行してアクセスすると。。。

```bash
$ curl http://localhost:3000
hello world!
```

ハロワでた！

# JSON で返す

次にレスポンスが json で返るようにしよう。

やり方は`response`オブジェクト（下記では`res`）に`Content-Type`を設定すれば良い。

```js
app.get('/', (req, res) => {
  res.header('Content-Type', 'application/json; charset=utf-8')
  res.send('{ "message": "hello world!" }')
})
```

これも実行すると。。。

```bash
$ node app
http://localhost:3000

$ curl http://localhost:3000
{ "message": "hello world!" }
```

json キタ！

# URI にパラメータをつける

パラメータありのリクエストにも対応しよう

`http://localhost:3000/<name>`として name に入れた名前で`hello <name>!`と返るようにしてみる。

以下のコードを追加する。

```js
app.get('/:name', (req, res) => {
  res.header('Content-Type', 'application/json; charset=utf-8')
  res.send({ message: `hello ${req.params.name}` })
})
```

get の URI 指定に`:<パラメータ>`と記述することでパラメータと認識され、`req.params.<パラメータ>`で取り出すことができる。

また、`res.send`にはオブジェクトをそのままぶち込める。これでもレスポンスは JSON にパースしてくれるので便利だ。

これを実行すると。。。

```bash
$ node app
http://localhost:3000

$ curl http://localhost:3000/hoge
{ "message": "hello hoge" }
```

イケるね！

# 参考

- [Node.js + Express で REST API 開発を体験しよう［作成編］ - Qiita](https://qiita.com/tamura_CD/items/e3abdab9b8c5aa35fa6b)

# 実行環境

- Node 8.11.3
- Express 4.16.4
