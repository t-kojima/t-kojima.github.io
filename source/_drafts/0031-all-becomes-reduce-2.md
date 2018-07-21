---
title: '全てがReduceになる(2)'
date: 2018-07-21 04:51:34
tags:
- javascript
- nodejs
---

[前回](https://t-kojima.github.io/2018/07/21/0030-all-becomes-reduce-1/)に続き、Array のメソッドを reduce で再実装していく。

- fill
- filter
- find
- findIndex

ちなみに[このリポジトリ](https://github.com/t-kojima/practice-all-becomes-reduce)でテストとかやってる。

<!-- more -->

# ルール

引き続き以下のルールでやっていく。

- for 文禁止
- Array#reduce を必ず使用する
- Array のインスタンスメソッドは reduce 以外使用不可
- Array のプロパティ、クラスメソッドは使用可能
- スプレッド構文は使用不可（reduce すら不要になる場合があるから）

# fill

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

# filter

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

# find

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

find の MDN は単に記述ミスかな？

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

# findIndex

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
