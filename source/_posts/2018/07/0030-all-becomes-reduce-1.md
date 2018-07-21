---
title: '全てがReduceになる(1)'
date: 2018-07-21 04:35:23
tags:
- javascript
- nodejs
---

ごめんなさいタイトルが言いたかっただけです。
それはそうと reduce をいじっていると**なんでもできそうな気がしてきた**ので、Array のメソッドを再実装してみる。

- concat
- copyWithin
- entries
- every

<!-- more -->

prototype 汚染なんか気にしないでガンガン上書きしちゃうぞ！（というかこういう機会じゃないと prototype 拡張なんかしないから…）

[Array - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array)のメソッドを上から順にやっつけていく。

# おさらい

はじめに reduce の仕様をおさらい。

<a href="https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

## 構文

```js
array.reduce(callback[, initialValue])
```

## 使用例（Sum）

```js
;[1, 2, 3].reduce(function(accumulator, currentValue, currentIndex, array) {
  return accumulator + currentValue
}, 0)
// > 6
```

## 説明

基本的にはレシーバとなる配列の反復処理を行う。callback は要素の数の分だけ呼び出されるが、その時各引数は以下の値を取る。

- accumulator : 前の要素の return 結果
- currentValue : 現在の要素値
- currentIndex : 現在の要素のインデクス
- array : レシーバ自身

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

ただ再実装するだけだと色々抜け道があるので以下のルールを規定する。

- for 文禁止
- Array#reduce を必ず使用する
- Array のインスタンスメソッドは reduce 以外使用不可
- Array のプロパティ、クラスメソッドは使用可能
- スプレッド構文は使用不可（reduce すら不要になる場合があるから）

# concat

[Array.prototype.concat() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/concat)

concat は二つの配列を結合し、新しい配列を返すメソッド。

## 構文

```js
var new_array = old_array.concat(value1[, value2[, ...[, valueN]]])
```

## 再実装

reduce を使って実装した結果がこれ

```js
Array.prototype.concat = function(...args) {
  const copy = Array.from(this)
  args.reduce((_, array) => {
    array.reduce((acc, cur) => {
      copy[copy.length] = cur
    }, null)
  }, null)
  return copy
}

const array = [1, 2, 3, 4]
console.log(array.concat([5, 6], [7, 8]))

// > [ 1, 2, 3, 4, 5, 6, 7, 8 ]
```

まず最初に非破壊的なメソッドなので copy する。arguments を使うとそのままでは Array#reduce できない（ESLint にも怒られるし）ので、レスト構文（`...args`)を使うことで reduce を呼べるようにしている。

次に copy 配列に args の配列を一つずつ追加していくが、Array#push が使えないので `array[array.length] = value` という形で順次配列を拡張していく。配列長を超えた要素に代入すると拡張されるのね、他の言語では大体 OutOfIndex 系のエラーが出るけど、これは助かる。

# copyWithin

[Array.prototype.copyWithin() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/copyWithin)

copyWithin は配列の指定の範囲を別の範囲に上書きする。

## 構文

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

また、以下の特性がある。

- target, start, end は Number の整数に変換される。
- 引数が負の場合は末尾から数えた位置（`this.length - x`）として扱われる。
- this 自身を変更し、this を返す。

逆に copyWithin は generic な関数として動作するとあるが、reduce が実装されていない型では動作できないので、この点は考慮しない。

## 再実装

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

copyWithin は動作確認に色々なパターンがあって面倒そうだったので[テストも書いてみた。](https://github.com/t-kojima/practice-all-becomes-reduce/blob/master/test/copy_within_test.js)

# entries

[Array.prototype.entries() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/entries)

配列の各要素のインデクスと値のペアを含むイテレータを返す。

## 構文

```js
array.entries()
```

基本的にイテレータそのものが必要になるシーンは普段無くて、大体そのまま for したり forEach したりするからこれも初めて使ったメソッドだった。

## 再実装

reduce で再実装すると以下のようになる。

```js
Array.prototype.entries = function() {
  const iterator = []
  let count = 0
  iterator.next = () => {
    try {
      return this.reduce(
        (acc, cur, index) =>
          count === index ? { value: [index, this[index]], done: false } : acc,
        { value: undefined, done: true }
      )
    } finally {
      count += 1
    }
  }
  return iterator
}
```

参考： [JavaScript の イテレータ を極める！ - Qiita](https://qiita.com/kura07/items/cf168a7ea20e8c2554c6)

イテレータカウントのインクリメントを try/catch でやっているのが若干苦しい感じがするが、reduce のループ内でやるともっと苦しそうだったからこうなっている。

それにしてもイテレータの再実装なんてのも初めてやったな…

# every

[Array.prototype.every() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/every)

指定された callback を要素が全てパスするかを判定する。

## 構文

```js
array.every(callback[, thisArg])
```

callback は以下 3 つの引数を取る。

- currentValue : 現在の要素値
- index : 現在の要素のインデクス
- array : レシーバ自身

また、いくつかの特性がある。

- thisArg が指定される場合、callback 内で this として扱われる。未指定の場合は undefined となる。
- 呼びだされた配列に破壊的変更を加えない。
- 空配列に対しては true を返す。

## 再実装

```js
Array.prototype.every = function(callback, thisArgs) {
  return this.reduce(
    (acc, cur, index, array) =>
      acc ? callback.call(thisArgs, cur, index, array) : acc,
    true
  )
}
```

call で関数スコープの this を指定できるのは初めて知った。しかし callback を function 式とアロー関数式で記述した時、this の評価が異なる点気になった。アロー関数式で記述した際は thisArgs が無視されるのだ。

ただ[MDN の this の注意書き](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/this)でも以下のように触れられているし

> 注: this 引数がアロー関数の実行時に call, bind, apply に渡されても無視されます。まだ call に引数を加えることはできますが、最初の引数(thisArg) は null をセットすべきです。.

every の polyfill にも以下のように call を使用した実装がされているので、そういう仕様で問題なさそうだ。

```js
// ii. testResult は、this 値としての T と、kValue、k、0 を含む引数リストを
//     ともなって、callbackfn の Call 内部メソッドを実行した結果です。
var testResult = callbackfn.call(T, kValue, k, O)
```

# さいごに

やばい、これ全部のメソッド書くとすさまじい長さになる

つづく
