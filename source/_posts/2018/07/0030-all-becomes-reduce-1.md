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

---

- 7/23 `concat`、`copyWithin`、`entries` を疎の配列を考慮した実装に修正

---

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

---

<h1 style="font-size: 4em;">concat</h1>

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

## 7/23 追記

**疎な配列**（`sparse array`）の考慮が漏れていた！

### 疎な配列とは

こういうやつ

```js
[,,]
// > [ <2 empty items> ]
Array(3)
// > [ <3 empty items> ]
[,,,3,4,,,]
// > [ <3 empty items>, 3, 4, <2 empty items> ]
```

性質として、length のみが設定された配列で、`empty item` となっている要素は undefined でもなく**undefined とは別の未定義**という状態だ。（便宜的に**疎の要素**とでも言おうか。。。）

こいつの厄介なところは、疎の要素は undefined ではないにも関わらず、アクセスして値を取り出すと undefined が取り出されてしまうので、一見区別がつかない点だ。判別するには `Object.prototype.hasOwnProperty` を使うか、もしくは `<property> in <object>` で判別するしかない。

また、疎な配列のコピー（シャローコピー）を行いたい場合にも、`Array.from(target)` は使用できない。疎の要素に undefined が入ってしまうのだ。

```js
Array.from([, , , , , ,])
// > [ undefined, undefined, undefined, undefined, undefined, undefined ]
```

`Array#concat` や `Array#slice` を使えば疎な配列のコピーを行うことができるが、concat を再実装する為に concat が必要だというのだから本末転倒だ。

そして concat（を含むほとんどのインスタンスメソッド）は疎な配列を正しく解釈する。

```js
Array(3).concat(Array(2))
// > [ <5 empty items> ]
```

なので頑張って対応するしかない。

### 改めて再実装

疎な配列に対応したものが以下の実装だ。

```js
Array.prototype.concat = function(...args) {
  const list = array => Array.from(Array(array.length))

  const pushValue = (from, to, index) => {
    if (index in from) {
      to[to.length] = from[index]
    } else {
      to.length += 1
    }
  }

  const copy = list(this).reduce((acc, cur, index) => {
    pushValue(this, acc, index)
    return acc
  }, [])

  const arrays = args.reduce((acc, cur) => {
    acc[acc.length] = Array.isArray(cur) ? cur : [cur]
    return acc
  }, [])

  arrays.reduce((_, array) => {
    list(array).reduce((acc, cur, index) => {
      pushValue(array, copy, index)
    }, null)
  }, null)
  return copy
}
```

#### 配列のコピー

まず配列のコピーはもともと

```js
const copy = Array.from(this)
```

としていたが、疎の配列を考慮することで以下のようになった。

```js
const list = array => Array.from(Array(array.length))

const pushValue = (from, to, index) => {
  if (index in from) {
    to[to.length] = from[index]
  } else {
    to.length += 1
  }
}

const copy = list(this).reduce((acc, cur, index) => {
  pushValue(this, acc, index)
  return acc
}, [])
```

`Array.from` でのコピーで疎の配列を扱えないのは前述の通りだが、reduce も反復処理の中で疎の要素を数えてくれない。なので必然的に index を基準とした要素アクセスを行う必要があり、そのために `Array.from(Array(array.length))` を reduce のレシーバとすることで、疎の要素があっても全ての要素が処理されるようにしている。

`pushValue` 関数では from がコピー元の配列、to がコピー先の配列で `to.push(from[index])` のような処理を（イメージです）行っている。`index in from` で疎の要素の判別を行い、コピー元が疎の要素であればコピー先に**疎の要素の追加**を行う。しかし疎の要素は通常代入できない。なのでコピー先の length を 1 つ拡張し、拡張された分の要素が疎の要素になるようにする。

#### 値の concat

これは単純に MDN の記載を見逃してただけなんだけど…concat は引数に配列だけではなく、値も取ることができる。

```js
const arrays = args.reduce((acc, cur) => {
  acc[acc.length] = Array.isArray(cur) ? cur : [cur]
  return acc
}, [])
```

なのでここでは引数をチェックし、値であれば配列でラップしている。配列かどうかの判定は `Array.isArray(cur)` の有無で判定する。

#### args(arrays) を concat

最後に引数の配列（arrays）を reduce で回し、更にその要素の配列を reduce で回して、copy にガンガン push していく。

```js
arrays.reduce((_, array) => {
  list(array).reduce((acc, cur, index) => {
    pushValue(array, copy, index)
  }, null)
}, null)
```

ここで回す配列も疎な配列の可能性がある為、`list` と `pushValue` 関数でのケアが必要だ。

疎の配列は完全に抜けてたなぁ…他のメソッドも全部見直さなきゃ…

---

<h1 style="font-size: 4em;">copyWithin</h1>

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

## 7/23 追記

疎な配列の対応をし、実装を見直した。

```js
// 長さが同じで要素がundefinedな配列を返す
// 疎の配列の場合reduce of empty array回避の為
const list = array => Array.from(Array(array.length))

// 配列(to)へ配列(from)のindex位置の要素をpushする。
// index位置が疎要素の場合はlengthを拡張して対応
// 疎要素を維持したままpushできる。
const pushValue = (from, to, index) => {
  if (index in from) {
    to[to.length] = from[index]
  } else {
    to.length += 1
  }
}

// 配列をコピーする。array.concat()と等価
// array#concatが利用できないのでreduceで実装
const clone = array =>
  list(array).reduce((acc, cur, index) => {
    pushValue(array, acc, index)
    return acc
  }, [])

Array.prototype.copyWithin = function(target, start = 0, end = this.length) {
  const parse = value =>
    Math.min(
      Math.max(Number.parseInt(value, 10) + (value < 0 ? this.length : 0), 0),
      this.length
    )

  const t = parse(target)
  const s = parse(start)
  const e = parse(end)

  const copy = clone(this)

  this.length = 0
  return list(copy).reduce((acc, cur, index) => {
    pushValue(
      copy,
      acc,
      index >= t && index < e - s + t ? s + index - t : index
    )
    return acc
  }, this)
}
```

`list`、`pushValue`、`clone` の関数は（恐らく）今後もそのまま使えると思うので、分かりやすく外だししてみた。

基本的には concat と同様の修正だ。配列のコピーは `Array.from` を使用せず、自作の `clone` 関数を使う。（concat の時は関数に `clone` という名前は付けていなかったが、同じロジックだ）

一点特筆すべきは `this.length = 0` の行だ。疎の要素を追加する際、既存の配列に対してインデクスを拡張しつつ追加することはできるが、length のサイズを変えずに既存の要素を疎の要素に置き換えることができない。しかし copyWithin は破壊的な操作になるので、配列のコピーを返すわけにもいかない。

そこで `this.length = 0` とすることで this の要素を全て削除し、改めてそこに要素を追加していく形にしている。これなら疎の要素も追加することができる。

あと地味に parse のロジックをワンライナーに書き直した。長いけど

---

<h1 style="font-size: 4em;">entries</h1>

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

## 7/23 追記

entries も疎の配列対応をした。

```js
Array.prototype.entries = function() {
  const list = array => Array.from(Array(array.length))
  const value = index => (index in this ? this[index] : undefined)

  const iterator = this[Symbol.iterator]()
  let count = 0
  iterator.next = () => {
    try {
      return list(this).reduce(
        (acc, cur, index) =>
          count === index ? { value: [index, value(index)], done: false } : acc,
        { value: undefined, done: true }
      )
    } finally {
      count += 1
    }
  }
  return iterator
}
```

### スプレッド構文対応

もともと `const iterator = []` となっていた所を、以下のように直した。

```js
const iterator = this[Symbol.iterator]()
```

`const iterator = []` でも `next()` で要素を取得する分には問題なく動いていたが、スプレッド構文で `[...array.entries()]` としたとき正常に配列を取得できなかった。`this[Symbol.iterator]()` で取得したイテレータを上書きする形にすることでちゃんと動くようになった。

### 疎の配列対応

entries でも疎の要素はちゃんと考慮されなければならない。疎の要素の場合、イテレータの要素は `{ value: undefined, done: false }` というオブジェクトになる。

対応なようは concat や copyWithin と同様だが、引数の parse や配列のコピーが不要なので、幾分か簡素にできている。

---

<h1 style="font-size: 4em;">every</h1>

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

[つづく](https://t-kojima.github.io/2018/07/22/0031-all-becomes-reduce-2/)

# 追記時の参考

- [JavaScript の配列のパターン | Web Scratch](https://efcl.info/2016/10/11/array-patterns/)
