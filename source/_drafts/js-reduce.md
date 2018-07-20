---
title: '全てがReduceになる'
date: 2018-07-19 16:24:23
tags:
- javascript
- node
---

ごめんなさいタイトルが言いたかっただけです。

それはそうと reduce をいじっている間になんでもできそうな気がしてきたので、Array のメソッドを再実装してみる。

<!-- more -->

prototype 汚染なんか気にしないでガンガン上書きしちゃうぞ！（というかこういう機会じゃないと prototype 拡張なんかしないから…）

[Array - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array)のメソッドを上から順にやっつけていく。

<!-- toc -->

# おさらい

はじめに reduce の仕様をおさらい。

<a href="https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

構文

```js
array.reduce(callback[, initialValue])
```

使用例（Sum）

```js
;[1, 2, 3].reduce(function(accumulator, currentValue, currentIndex, array) {
  return accumulator + currentValue
}, 0)
// > 6
```

基本的にはレシーバとなる配列の反復処理を行う。callback は要素の数の分だけ呼び出されるが、その時各引数は以下の値を取る。

- accumulator : 前の要素の return 結果
- currentValue : 現在の要素値
- currentIndex : 現在の要素のインデクス
- array : レシーバの配列

最後の要素の return 結果が、最終的な reduce の戻り値として返される。
また、最初の要素の accumulator は initialValue の値となり、initialValue を省略した場合は accumulator が最初の要素、currentValue が二番目の要素になる。

ちなみに上記の例は以下の流れで処理される

| index | callback           | return |
| ----- | ------------------ | ------ |
| 0     | (0, 1, 0, [1,2,3]) | 0 + 1  |
| 1     | (1, 2, 1, [1,2,3]) | 1 + 2  |
| 2     | (3, 3, 2, [1,2,3]) | 3 + 3  |

最後の要素の return が 6 なので、ちゃんと sum として動いていることがわかる。

# ルール

- for 文禁止
- Array#reduce を必ず使用する
- Array のインスタンスメソッドは reduce 以外使用不可
- Array のプロパティ、クラスメソッドは使用可能
- スプレッド構文は使用不可（reduce すら不要になる場合があるから）

# concat

[Array.prototype.concat() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/concat)

concat は二つの配列を結合し、新しい配列を返すメソッド。

構文

```js
var new_array = old_array.concat(value1[, value2[, ...[, valueN]]])
```

reduce のみで実装した結果がこれ

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

// > [ 1, 2, 3, 4, 5, 6, 7, 8 ]
```

- reduce をただの for として使う
- 非破壊的なメソッドなので copy する

単に for ができれば実装できる。カンタン

# copyWithin

[Array.prototype.copyWithin() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/copyWithin)

copyWithin は配列の指定の範囲を別の範囲に上書きする。

構文

```js
arr.copyWithin(target[, start = 0[, end = this.length]])
```

初めて見たメソッドなんだけど、MDN の説明も訳わかんなくて理解に時間が掛かった。

```js
;[1, 2, 3, 4, 5, 6, 7].copyWithin(0, 3, 5)
//         ~~~~~この範囲を
//====ここに上書きする
// > [ 4, 5, 3, 4, 5, 6, 7 ] こうなる
```

ちょっと気になる点として、引数の end はコピー対象の末尾のインデクスを指定しているわけではなく、end-1 が実際コピーされる末尾インデクスになる。なので上の例だと start/end を `3 to 5` で指定しているので、`4,5,6` がコピー対象と思いがちだが、実際は `4,5` のみがコピー対象だ。

また、以下の動作も仕様となる。

- target, start, end は Number の整数に変換される。
- 引数が負の場合は末尾から数えた位置（`this.length - x`）として扱われる。
- this 自身を変更し、this を返す。

逆に copyWithin は generic な関数として動作するとあるが、reduce が実装されていない型では動作できないので、この点は考慮しない。

これを reduce で実装するとこうなる

```js
Array.prototype.copyWithin = function(target, start = 0, end = this.length) {
  const copy = Array.from(this)
  const parse = value => {
    if (value > this.length) return this.length
    if (value + this.length < 0) return 0
    return value < 0
      ? this.length + Number.parseInt(value, 10)
      : Number.parseInt(value, 10)
  }
  const t = parse(target)
  const s = parse(start)
  const e = parse(end)

  return this.reduce((acc, cur, index) => {
    if (index >= t && index < e - s + t) {
      acc[index] = copy[s + index - t]
    }
    return acc
  }, this)
}

const array = [1, 2, 3, 4, 5, 6, 7]
console.log(array.copyWithin(0, 3, 5))
```

中身は多く感じるが、大半は整数へのパースと配列長を超えた引数やマイナス引数の補正だ。

いくつかポイントを解説

```js
const copy = Array.from(this)
```

後の reduce 内の処理で this を書き換えるてしまうので、コピー元の配列を保存しておく

```js
const parse = value => {
  if (value > this.length) return this.length
  if (value + this.length < 0) return 0
  return value < 0
    ? this.length + Number.parseInt(value, 10)
    : Number.parseInt(value, 10)
}
const t = parse(target)
const s = parse(start)
const e = parse(end)
```

parse を通すと以下の処理を行った値になる。

- 絶対値が配列長を超えていてプラス値の場合、配列長に丸める。
- 絶対値が配列長を超えていてマイナス値の場合、0 に丸める。
- 値がマイナスの場合、配列長を足して末尾から数えた位置を返す。
- 値が少数の場合、整数に丸める（切り捨て）

```js
return this.reduce((acc, cur, index) => {
  if (index >= t && index < e - s + t) {
    acc[index] = copy[s + index - t]
  }
  return acc
}, this)
```

target が index 以上、かつ `target` + `startとendの差分`が index 未満の場合、該当 index の要素に `start` + `targetとindexの差分`をコピーする。

# entries

[Array.prototype.entries() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/entries)

配列の各要素のインデクスと値のペアを含むイテレータを返す。
