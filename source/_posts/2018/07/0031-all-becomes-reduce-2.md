---
title: '全てがReduceになる(2)'
date: 2018-07-22 21:51:34
tags:
- javascript
- nodejs
---

[前回](https://t-kojima.github.io/2018/07/21/0030-all-becomes-reduce-1/)に続き、Array のメソッドを reduce で再実装していく。

- fill
- filter
- find
- findIndex
- forEach
- includes

ちなみに[このリポジトリ](https://github.com/t-kojima/practice-all-becomes-reduce)でテストとかやってる。

<!-- more -->

# ルール

引き続き以下のルールでやっていく。

- for 文禁止
- Array#reduce を必ず使用する
- Array のインスタンスメソッドは reduce 以外使用不可
- Array のプロパティ、クラスメソッドは使用可能
- スプレッド構文は使用不可（reduce すら不要になる場合があるから）

---

<h1 style="font-size: 4em;">fill</h1>

[Array.prototype.fill() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/fill)

fill は指定した値で配列を埋めるメソッド、0 埋めとかによく使う。

## 構文

```js
array.fill(value[, start = 0[, end = this.length]])
```

引数の特性は copyWithin とよく似ている。

- start から end の値を置き換えるが、end は含まない。
- start と end はマイナスの場合、start+length もしくは end+length として扱う。（末尾から数えた位置）
- start, end は Number の整数に変換される。
- this 自身を変更し、this を返す。

## 再実装

```js
Array.prototype.fill = function(target, start = 0, end = this.length) {
  const parse = value => {
    if (value > this.length) return this.length
    if (value + this.length < 0) return 0
    return Number.parseInt(value, 10) + (value < 0 ? this.length : 0)
  }
  const s = parse(start)
  const e = parse(end)

  return this.reduce((acc, cur, index) => {
    if (index >= s && index < e) {
      acc[index] = target
    }
    return acc
  }, this)
}
```

parse 関数は copyWithin の時に比べ少しだけ変えた。とはいえ最後の return 部分を若干 DIY にしただけなので、処理内容自体は全く同じだ。そして parse の対象は start と end のみとなる。

fill される条件は単純にインデクスが start 以上、end 未満の場合だ。ここは単純になっていて分かりやすい。

## 7/24 追記

疎な配列の対応をし、実装を見直した。

```js
Array.prototype.fill = function(target, start = 0, end = this.length) {
  const list = array => Array.from(Array(array.length))
  const parse = value =>
    Math.min(
      Math.max(Number.parseInt(value, 10) + (value < 0 ? this.length : 0), 0),
      this.length
    )

  const s = parse(start)
  const e = parse(end)

  return list(this).reduce((acc, cur, index) => {
    if (index >= s && index < e) {
      acc[index] = target
    }
    return acc
  }, this)
}
```

基本的にはレシーバとなる this を list 関数でラップし、疎の要素があっても列挙されるようにした。あと copyWithin と同様に parse をワンライナーに修正。

---

<h1 style="font-size: 4em;">filter</h1>

[Array.prototype.filter() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/filter)

filter は C#でいう Linq の Where、Ruby でいう Array#select で、非常に良く使われるメソッドだ。利用率としては map か filter か、といったところだろう。

callback の判定で true を返す要素のみを新たな配列として返す。

## 構文

```js
var newArray = array.filter(callback[, thisArg])
```

callback は以下 3 つの引数を取る。

- element : 現在の要素値
- index : 現在の要素のインデクス
- array : レシーバ自身

また、いくつかの特性がある。この辺り every とよく似ている。

- thisArg が指定される場合、callback 内で this として扱われる。未指定の場合は undefined となる。
- 呼びだされた配列に破壊的変更を加えない。
- callback の判定が全て false の場合、空配列を返す。

## 再実装

```js
Array.prototype.filter = function(callback, thisArgs) {
  return this.reduce((acc, cur, index, array) => {
    if (callback.call(thisArgs, cur, index, array)) {
      acc[acc.length] = cur
    }
    return acc
  }, [])
}
```

実装は非常にシンプルになった。reduce の初期値として空配列を指定し、callback に通った要素だけ追加していけばいい。

---

<h1 style="font-size: 4em;">find</h1>

[Array.prototype.find() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/find)

find は callback の判定をパスした最初の要素を返す。これもよく使われるね。

## 構文

```js
array.find(callback[, thisArg])
```

callback は以下 3 つの引数を取る。filter と全く同じだ。

- element : 現在の要素値
- index : 現在の要素のインデクス
- array : レシーバ自身

同様に以下の特性がある。

- thisArg が指定される場合、callback 内で this として扱われる。未指定の場合は undefined となる。
- 呼びだされた配列に破壊的変更を加えない。
- callback の判定が全て false の場合（見つからなかった時）、undefined を返す。

### thisArg について

余談だが、thisArg を指定しない場合（または undefined を指定した場合）、callback 内で呼び出される this は undefined として扱われる…と MDN には書いてあるが、実際はそうはならない。

MDN の filter の場合は、thisArg について以下のように記述されている。

> filter に thisArg パラメータが与えられると、そのオブジェクトは callback 関数内の this として使用されます。thisArg が与えられないか undefined の場合、callback 関数内の this は関数内の this を決定する一般的ルールに従って決められます。

この記述は正しくて、実際このように動作する。

しかし MDN の find には以下のように記述されている。

> find に thisArg 引数を与えた場合、各 callback の呼び出し時に、その与えたオブジェクトが、this として使用されます。この引数を省略した場合、this は undefined になります。

単に undefined になると書いてある。が、実際の挙動は filter と同様に**関数内の this を決定する一般的なルールに従って決められる。**

これは恐らく `call` に `this` を渡した際の共通の挙動と思われるので、他の thisArg を引数に取る関数も同様の動作になるはずだ。

MDN の find は単に記述ミスかな？

## 再実装

```js
Array.prototype.find = function(callback, thisArgs) {
  return this.reduce(
    (acc, cur, index, array) =>
      acc === undefined && callback.call(thisArgs, cur, index, array)
        ? cur
        : acc,
    undefined
  )
}
```

reduce の初期値に undefined を指定し、callback の判定をパスしたら accumulator に要素を代入する。すると以後の callback は評価されず、最初に代入した要素がそのまま返される。

---

<h1 style="font-size: 4em;">findIndex</h1>

[Array.prototype.findIndex() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/findIndex)

find のインデクスを返す版だ。

## 構文

```js
array.findIndex(callback[, thisArg])
```

callback の引数や、その他の特性は find と全く一緒だ。唯一見つからなかった場合の戻り値のみが異なる。

- callback の判定が全て false の場合（見つからなかった時）、-1 を返す。

## 再実装

```js
Array.prototype.findIndex = function(callback, thisArgs) {
  return this.reduce(
    (acc, cur, index, array) =>
      acc < 0 && callback.call(thisArgs, cur, index, array) ? index : acc,
    -1
  )
}
```

再実装のコードも find とほぼほぼ同じだね。初期値が-1 であることと、要素の代わりに index を返すことくらいか…

## 7/24 追記

疎な配列の対応をし、実装を見直した。

```js
Array.prototype.findIndex = function(callback, thisArgs) {
  return Array.from(this).reduce(
    (acc, cur, index, array) =>
      acc < 0 && callback.call(thisArgs, cur, index, array) ? index : acc,
    -1
  )
}
```

findIndex には疎の要素を undefined として判別する特徴がある。つまり以下のようなコードがあった場合、戻り値は 3 になる。

```js
;[0, 1, 2, , , , ,].findIndex(v => v === undefined)
// > 3
```

そのため、`Array.from(this)`とすることで元の配列の疎の要素を全て undefined に変換をしている。

```js
Array.from([0, 1, 2, , , , ,]) // を通すと
// > [0,1,2,undefined,undefined,undefined,undefined]
// 疎の要素はundefinedになる
```

# flatMap と flatten

[Array.prototype.flatMap() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/flatMap)
[Array.prototype.flat() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/flat)

こいつらは Node.js で利用できない＆代替手段として既に reduce が例示されているのでやらないです。。。

---

<h1 style="font-size: 4em;">forEach</h1>

[Array.prototype.forEach() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/forEach)

配列の反復処理を行う。本質的には for と同じだが、for は文で forEach は式といった違いはある。

## 構文

```js
array.forEach(function callback(currentValue[, index[, array]]) {
    //your iterator
}[, thisArg]);
```

なぜか構文の書き方が他のメソッドと違う…今までのパターンでいくと `array.forEach(callback[, thisArg])` になるはず。どうでもいいけど…

構文にある通り、callback は以下 3 つの引数を取る。

- currentValue : 現在の要素値
- index : 現在の要素のインデクス
- array : レシーバ自身

そして以下の特性を持つ。

- thisArg が指定される場合、callback 内で this として扱われる。未指定の場合は undefined となる。
- 呼びだされた配列に破壊的変更を加えない。
- 戻り値は undefined となる。

## 再実装

```js
Array.prototype.forEach = function(callback, thisArgs) {
  return this.reduce((acc, cur, index, array) => {
    callback.call(thisArgs, cur, index, array)
    return acc
  }, undefined)
}
```

ただ渡された callback を実行するだけの素直な実装だ。しいて言うなら undefined を返すよう accumulator は常に undefined であることくらいか。

---

<h1 style="font-size: 4em;">includes</h1>

[Array.prototype.includes() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/includes)

指定の要素が配列に含まれるかを調べる。

## 構文

```js
array.includes(searchElement[, fromIndex])
```

findIndex は検索を開始する位置（インデクス）だが、以下の性質を持つ。

- 省略時は 0
- 配列の長さ以上の場合、includes は無条件に false を返す。
- 負の値の場合、配列を逆から数えた値となる。（補正される）
- 補正された負の値が 0 以下になった場合、0 として扱われる。

また、includes は `Array-like` なオブジェクト（`arguments`など）でも利用可能だが、`Array-like`なオブジェクトではそもそも reduce が使えないので、今回は考慮しないこととにする。

## 再実装

```js
Array.prototype.includes = function(target, fromIndex = 0) {
  const parse = value => {
    if (value > this.length) return this.length
    if (value + this.length < 0) return 0
    return Number.parseInt(value, 10) + (value < 0 ? this.length : 0)
  }
  const findex = parse(fromIndex)

  return this.reduce(
    (acc, cur, index) =>
      !acc &&
      index >= findex &&
      (cur === target || (Number.isNaN(cur) && Number.isNaN(target)))
        ? true
        : acc,
    false
  )
}
```

parse 関数は copyWithin や fill で使ったものと同じだ。これを findIndex に適用する。

target と currentItem(cur)は `===` (厳格な比較)で比較し、合致していた場合 true となるが、`NaN` の場合は `===` で比較できないので別途 `isNaN` での比較としている。

[等価性の比較とその使いどころ - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Equality_comparisons_and_when_to_use_them) を参照しても、`===` で `NaN` のみ true にならないとしているので、`NaN` のみ例外ケースとして問題ないだろう。

> (x !== x) が true になる唯一のケースは x が NaN である場合です。

## 7/24 追記

疎な配列の対応をし、実装を見直した。

```js
Array.prototype.includes = function(target, fromIndex = 0) {
  const list = array => Array.from(Array(array.length))

  const parse = value =>
    Math.min(
      Math.max(Number.parseInt(value, 10) + (value < 0 ? this.length : 0), 0),
      this.length
    )

  const sameValueZero = (v1, v2) =>
    v1 === v2 || (Number.isNaN(v1) && Number.isNaN(v2))

  const findex = parse(fromIndex)

  return list(this).reduce(
    (acc, cur, index) =>
      !acc && index >= findex && sameValueZero(this[index], target)
        ? true
        : acc,
    false
  )
}
```

他のメソッドと同様に、this を list 関数でラップする対応を行った。

また、疎な配列とは関係ないが、SameValueZero 判定を関数に切り出し明確にした。

# おわりに

29 メソッドあるうち 10 メソッドできた。まだまだあるな…

つづく
