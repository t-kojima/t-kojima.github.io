---
title: js-reduce
date: 2018-07-19 16:24:23
tags:
---

どこかで「reduce はチューリング完全」という記事を見かけた気がしたが記憶が定かではない…
それはそうと reduce をいじっている間になんでもできそうな気がしてきたので、Array のメソッドを再実装してみる。

<!-- more -->

prototype 汚染なんか気にしないでガンガン上書きしちゃうぞ！（というかこういう機会じゃないと prototype 拡張なんかしないから…）

[Array - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array)のメソッドを上から順にやっつけていく。

# concat

- reduce をただの for として使う
- 非破壊的なメソッドなので copy する

```js
Array.prototype.concat = function(array) {
  const copy = Array.from(this)
  array.reduce((acc, cur) => {
    copy.push(cur)
  }, [])
  return copy
}

const array = [1, 2, 3, 4]
console.log(array.concat([5, 6, 7, 8]))
```
