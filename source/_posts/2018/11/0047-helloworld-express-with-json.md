---
title: '[Node] Expressを使って最速でJSONを返す'
date: 2018-11-02 17:13:38
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
const express = require('express');
const app = express();

app.get('/', (req, res) => res.send('hello world!'));

app.listen(3000, () => console.log('http://localhost:3000'));
```

うーん、4 行！かんたん

最後の`console.log`は無くてもいいが、アドレスを入れておくと起動後に即開けるので楽ちん

```js
$ node app
http://localhost:3000
```

実行してブラウザアクセスすると。。。

```bash
> hello world!
```

ハロワでた！

## 実行環境

- Node 8.11.3
- Express 4.16.4
