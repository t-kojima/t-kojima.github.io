---
title: '全てがReduceになる(4)'
date: 2018-07-26 23:09:28
tags:
- javascript
- nodejs
---

[前回](https://t-kojima.github.io/2018/07/24/0032-all-becomes-reduce-3/)に続き、Array のメソッドを reduce で再実装していく。なんかだんだんダレてきたけど、もう少しだ。

- reduceRight
- reverse
- shift
- slice
- some

[実装及びテストのリポジトリ(practice-all-becomes-reduce)](https://github.com/t-kojima/practice-all-becomes-reduce)

<!-- more -->

# ルール

引き続き以下のルールでやっていく。

- for 文禁止
- Array#reduce を必ず使用する
- Array のインスタンスメソッドは reduce 以外使用不可
- Array のプロパティ、クラスメソッドは使用可能
- スプレッド構文は使用不可（reduce すら不要になる場合があるから）

---

<h1 style="font-size: 4em;">reduceRight</h1>

[Array.prototype.reduceRight() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/reduceRight)

reduceRight は通常左の要素から順に処理する reduce を右の要素から処理するようにしたもの。

## 構文

```js
result = array.reduceRight(callback[, initialValue]);
```

callback は以下 4 つの引数を取る。

- previousValue : コールバックの戻り値の累積。reduce の accumulator
- currentValue : 現在処理されている配列の要素
- index : 現在処理されている要素のインデックス
- array : レシーバとなる配列

initialValue を指定すると、それが最初の要素を処理する際の previousValue となる。逆に initialValue が未指定の場合、最初の要素が previousValue となり、インデックスは 1 から始まる。

そして全ての要素の処理が完了した時、最後の戻り値が reduceRight の戻り値となる。

しかしこの callback の引数 array は全く使ったことがないな…多分こういう呼び出し方をする時使うんだろうけど、こういう呼び出し方しないからなぁ

```js
[1,2,3,4].reduceRight((acc, cur, index, array) => ...)
```

## 再実装

reduce で reduce を作るなんて頭がフットーしそうだよう

```js
Array.prototype.reduceRight = function(...args) {
  const reverse = Array.from(this).reduce((acc, cur, index) => {
    if (index in this) {
      acc[this.length - index - 1] = cur
    }
    return acc
  }, Array(this.length))

  const callback = (acc, cur, index) =>
    args[0].call(this, acc, cur, this.length - index - 1, this)

  return args.length > 1
    ? reverse.reduce(callback, args[1])
    : reverse.reduce(callback)
}
```

それはそうと、元の配列を反転させてから reduce させる形にした。

initialValue は引数として定義してしまうと純粋な未指定にできない（引数を定義すると未指定にしても undefined になってしまい、initialValue 無しの動作にはならないから）ので、可変引数で受け取り長さをチェックすることで受け取れるようにしている。

あとは callback 内でインデックスを反転させればいいだけだ。

---

<h1 style="font-size: 4em;">reverse</h1>

[Array.prototype.reverse() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/reverse)

## 構文

```js
a.reverse()
```

戻り値として反転された配列を返すが、レシーバとなる配列も破壊的に反転される。

## 再実装

```js
Array.prototype.reverse = function() {
  const copy = Array.from(this).reduce((acc, cur, index) => {
    if (index in this) {
      acc[index] = cur
    }
    return acc
  }, Array(this.length))

  this.length = 0
  this.length = copy.length

  return Array.from(copy).reduce((acc, cur, index) => {
    if (index in copy) {
      acc[copy.length - index - 1] = cur
    }
    return acc
  }, this)
}
```

反転する処理自体は reduceRight でもやったのでそのまま流用する。ただしレシーバ自身を破壊的に変更するため、一旦コピーの配列を用意し、それを利用して自身を書き換える。

その為 `this.length = 0` で一旦レシーバ自身をリセットし、改めて `this.length = copy.length` で疎な配列に作り変えている。代入が可能であれば `this = Array(this.length)` に等しい処理だが、this への代入は許されないのでこのような処理になる。

ただモロに手続き型の記述になっているし、コードだけだと意図が不明瞭なのでちょっとイケてない書き方だなーと思っている。まあ無理やり reduce で処理している以上意図なんて不明瞭もクソもないんだけど…

---

<h1 style="font-size: 4em;">shift</h1>

[Array.prototype.shift() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/shift)

配列の最初の要素を取り除き、その要素を返す。pop の先頭版のようなものだ。

## 構文

```js
arr.shift()
```

戻り値として先頭の要素を返し、元の配列から削除する。空の配列で shift した場合、undefined が返る。

## 再実装

```js
// Array.prototype.shift = にするとmochaが循環する。
Array.prototype.shift2 = function() {
  const copy = Array.from(this).reduce((acc, cur, index) => {
    if (index in this) {
      acc[index] = cur
    }
    return acc
  }, Array(this.length))

  this.length = 0
  this.length = copy.length ? copy.length - 1 : 0

  return Array.from(this).reduce((acc, cur, index) => {
    if (index + 1 in copy) {
      this[index] = copy[index + 1]
    }
    return index === 0 ? copy[index] : acc
  }, undefined)
}
```

ロジックとは関係ない（？）んだけど、Array.prototype.shift をそのまま上書きすると mocha のテストが循環する自体に陥った。恐らく describe や it の処理中で shift を使っていたのだろう。正確な原因は掴めなかったが、このメソッドのみ `shift2` とすることで単純な上書きを避ける実装にした。

pop と違い、先頭の要素を切り出すという処理に結構苦心してしまった。コピー配列を作り、元の配列は長さを 1 切り詰める。そしてコピー配列の要素をインデックスを一つずらして元の配列に詰めなおして行くという流れだ。

ここでも this を破壊的に書き換える為、length を操作している。ちなみに length をリセットせずに this を書き換えようとしても、プロパティ無しの要素を再現できないため、length をリセットして全ての要素を一度プロパティ無しにする必要がある。

---

<h1 style="font-size: 4em;">slice</h1>

[Array.prototype.slice() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/slice)

slice は配列の一部を切り出すメソッドだ。配列全部を切り出してコピーする用途に使われることが多い。

## 構文

```js
arr.slice([begin[, end]])
```

begin は切り出す開始インデックスを指定し、end は切り出す終了インデックス（の次のインデックス）を指定する。2 と 4 を指定した場合、2 と 3 のインデックスの要素が配列として返される。

今までのインデックス指定のメソッドがそうだったように、begin、end もマイナス値を許容し、指定された場合配列の終わりからの位置になる。

## 再実装

```js
Array.prototype.slice = function(begin = 0, end = this.length) {
  const parse = value =>
    Math.min(
      Math.max(Number.parseInt(value, 10) + (value < 0 ? this.length : 0), 0),
      this.length
    )

  const b = parse(begin)
  const e = parse(end)

  const copy = Array.from(this).reduce((acc, cur, index) => {
    if (index in this) {
      acc[index] = cur
    }
    return acc
  }, Array(this.length))

  return Array.from(this).reduce((acc, cur, index) => {
    if (index >= b && index < e) {
      acc[acc.length] = copy[index]
    }
    return acc
  }, [])
}
```

再実装自体は何も難しいことはしてない。parse 関数や copy の処理も今まで散々出てきたものだ。そしてメインの reduce では begin 以上・end 以下のインデックスの要素を配列として返している。

---

<h1 style="font-size: 4em;">some</h1>

[Array.prototype.some() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/some)

配列の要素に対して渡した関数で検証を行い、１つでも検証をパスするかを判定するメソッドだ。

## 構文

```js
arr.some(callback[, thisArg])
```

callback の引数は以下の 3 つ、このパターンもいい加減飽きたな

- currentValue : 現在の要素値
- index : 現在の要素のインデクス
- array : レシーバ自身

そして以下の特徴を持つ。

- thisArg が指定される場合、callback 内で this として扱われる。未指定の場合は undefined となる。
- callback は値が代入されている配列のインデックスに対してのみ呼び出される。（疎の要素はスキップされる）
- 呼びだされた配列に破壊的変更を加えない。
- callback に対していずれかの要素が truhly を返したら戻り値は true、それ以外の場合は false

## 再実装

```js
Array.prototype.some = function(callback, thisArg) {
  return this.reduce(
    (acc, cur, index, array) =>
      !acc ? !!callback.call(thisArg, cur, index, array) : acc,
    false
  )
}
```

はい、every とほとんど同じですね。ちなみに every はこれ。

```js
Array.prototype.every = function(callback, thisArgs) {
  return this.reduce(
    (acc, cur, index, array) =>
      acc ? !!callback.call(thisArgs, cur, index, array) : acc,
    true
  )
}
```

なんかもう間違い探しみたいだけど、initialValue が `true/false` の違いと、その違いによる継続条件の判定が `acc` と `!acc` になっている点のみが違う。

# おわりに

実は sort まで再実装しているんだけど、sort の内容がすごい長くなりそうなので一旦ここで切る。

つづく
