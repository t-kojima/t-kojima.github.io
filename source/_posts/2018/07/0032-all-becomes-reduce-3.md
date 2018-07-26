---
title: '全てがReduceになる(3)'
date: 2018-07-24 23:19:46
tags:
- javascript
- nodejs
---

[前回](https://t-kojima.github.io/2018/07/22/0031-all-becomes-reduce-2/)に続き、Array のメソッドを reduce で再実装していく。

- indexOf
- join
- keys
- lastIndexOf
- map
- pop
- push

[実装及びテストのリポジトリ(practice-all-becomes-reduce)](https://github.com/t-kojima/practice-all-becomes-reduce)

<!-- more -->

---

- 7/25 `pop` を微修正

---

# ルール

引き続き以下のルールでやっていく。

- for 文禁止
- Array#reduce を必ず使用する
- Array のインスタンスメソッドは reduce 以外使用不可
- Array のプロパティ、クラスメソッドは使用可能
- スプレッド構文は使用不可（reduce すら不要になる場合があるから）

---

<h1 style="font-size: 4em;">indexOf</h1>

[Array.prototype.indexOf() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/indexOf)

indefOf は引数の要素と同値の要素のインデックスを返す。厳密な同値（`===`）での比較となるため、複雑な条件指定ができないので、findIndex などに比べると若干使い辛い。

## 構文

```js
var index = array.indexOf(searchElement[, fromIndex]);
```

- 配列に searchElement と同値の要素がある場合、そのインデックスを返す。
- 同値の要素が存在しない場合、-1 を返す。
- fromIndex を指定した場合、fromIndex 以降のインデックスを検索対象とする。
- fromIndex を省略した場合、0 として扱う。
- fromIndex が配列長以上の場合、検索はされずに-1 が返される。
- fromIndex が負の値の場合、配列を逆からのオフセットとされる。

## 再実装

以下のように reduce で再実装する。

```js
Array.prototype.indexOf = function(target, fromIndex = 0) {
  const parse = value =>
    Math.min(
      Math.max(Number.parseInt(value, 10) + (value < 0 ? this.length : 0), 0),
      this.length
    )

  const findex = parse(fromIndex)

  return this.reduce(
    (acc, cur, index) =>
      acc === -1 && index >= findex && cur === target ? index : acc,
    -1
  )
}
```

parse 関数は今までと同じやつだ。その他 fromIndex 以降を検索していたり、厳密な同値で要素比較をする点など、仕様通りの実装となっている。

疎な配列の場合の考慮がされていないが、疎の要素は searchElement に指定のしようが無いので検索対象に含まれなくても問題ない。なので `this.reduce` で疎の要素が全てスキップされても大丈夫なわけだ。

---

<h1 style="font-size: 4em;">join</h1>

[Array.prototype.join() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/join)

join は配列の要素を結合して文字列を返すメソッドだ。

## 構文

```js
str = arr.join([(separator = ',')])
```

- 要素を結合する際のセパレータはデフォルトで `,` だが、任意に指定することができる。
- undefined と null は空文字に変換される。

## 再実装

```js
Array.prototype.join = function(separator = ',') {
  const value = v => (v === undefined || v === null ? '' : v)

  // 空配列は空文字列を返す
  if (this.length === 0) return ''

  return Array.from(this).reduce(
    (acc, cur) => value(acc) + separator + value(cur)
  )
}
```

まず空配列（`length === 0`）の場合は空文字（`''`）が返される。これは MDN には記載が無いが、[ECMAScript 2015](https://www.ecma-international.org/ecma-262/6.0/#sec-array.prototype.join)には記載されている。

> 8.  If len is zero, return the empty String.

そして疎の要素は undefined に変換する必要がある。恐らく以下の Get で要素を取得しているので undefined になっているのだろう。実際にオリジナルの join はそのような動きをする。

> 9.  Let element0 be Get(O, "0").

その為、`Array.from(this)` を経由することで疎の要素を undefined に変換している。あとは undefined と null を空文字に変換して結合するだけだ。

---

<h1 style="font-size: 4em;">keys</h1>

[Array.prototype.keys() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/keys)

keys は配列インデックスのイテレータを返す。

## 構文

```js
arr.keys()
```

イテレータオブジェクトを戻り値として戻す。

## 再実装

```js
Array.prototype.keys = function() {
  const list = array => Array.from(Array(array.length))

  const iterator = this[Symbol.iterator]()
  let count = 0
  iterator.next = () => {
    try {
      return list(this).reduce(
        acc => (count < this.length ? { value: count, done: false } : acc),
        { value: undefined, done: true }
      )
    } finally {
      count += 1
    }
  }
  return iterator
}
```

entries はインデックス+値だったの対し、keys はインデックスだけなので、entries を簡単にした感じだ。

count 変数はインデックスを兼ねていて、length 以下なら `{ value: count, done: false }` を返す形だ。ただ疎の要素をスキップしてしまうとインデックスが歯抜けになってしまうので、他のメソッド同様に this を list 関数でラップする。

---

<h1 style="font-size: 4em;">lastIndexOf</h1>

[Array.prototype.lastIndexOf() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/lastIndexOf)

lastIndexOf は IndexOf の逆から検索する版

## 構文

```js
var index = array.lastIndexOf(searchElement[, fromIndex]);
```

- 配列に searchElement と同値の要素がある場合、そのインデックスを返す。
- 同値の要素が存在しない場合、-1 を返す。
- fromIndex を指定した場合、fromIndex 以降のインデックスを検索対象とする。
- fromIndex を省略した場合、配列の長さをデフォルトとして扱う。
- fromIndex が負の値で絶対値が配列長以上の場合、検索はされずに-1 が返される。
- fromIndex が負の値の場合、配列を逆からのオフセットとされる。

## 再実装

```js
Array.prototype.lastIndexOf = function(target, fromIndex = this.length) {
  const parse = value =>
    Math.min(
      Math.max(Number.parseInt(value, 10) + (value < 0 ? this.length : 0), -1),
      this.length
    )

  const findex = parse(fromIndex)

  return this.reduce(
    (acc, cur, index) => (index <= findex && cur === target ? index : acc),
    -1
  )
}
```

ほとんど indexOf と同じである。indexOf は最初の要素が見つかったら以後の要素はスキップしていたが、lastIndexOf では要素が見つかっても検索を止めず、最後まで検索して最後に見つかった要素を返すようにしている。

また、地味に parse 関数か返す最低値が-1 になっている。これは最低値が 0 だと最初の要素を検索対象としてしまうため、「負の値で絶対値が配列長以上の場合」は-1 になり、全ての要素が検索対象外となるようにしている。

---

<h1 style="font-size: 4em;">map</h1>

[Array.prototype.map() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/map)

皆さんおなじみの map だ。map は全ての要素に何かしらの処理（callback）を掛け、結果を返すメソッドだ。

## 構文

```js
var new_array = arr.map(function callback(currentValue[, index[, array]]) {
    // 新しい配列の要素を返す
}[, thisArg])
```

callback の引数は以下の 3 つ、いつものやつだ

- currentValue : 現在の要素値
- index : 現在の要素のインデクス
- array : レシーバ自身

そして以下の特徴を持つ。

- thisArg が指定される場合、callback 内で this として扱われる。未指定の場合は undefined となる。
- callback は値が代入されている配列のインデックスに対してのみ呼び出される。（疎の要素はスキップされる）
- 呼びだされた配列に破壊的変更を加えない。

## 再実装

```js
Array.prototype.map = function(callback, thisArgs) {
  const list = array => Array.from(Array(array.length))

  const push = (acc, cur, index, array) => {
    if (index in array) {
      acc[acc.length] = cur
    } else {
      acc.length += 1
    }
  }

  return list(this).reduce((acc, cur, index, array) => {
    push(acc, callback.call(thisArgs, this[index], index, array), index, this)
    return acc
  }, [])
}
```

map は疎の要素をスキップして良いのだが、戻り値の配列を組み直さなければならないので、やはり this を list 関数でラップして、インデックス全てを走査する必要がある。

あとは callback の結果を順次配列に詰め込み、reduce の戻り値として返すだけだ。

---

<h1 style="font-size: 4em;">pop</h1>

[Array.prototype.pop() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/pop)

配列の末尾の要素を取得し、配列からはその要素を削除する。push と対になっていてスタック的な動作が可能になる。

## 構文

```js
arr.pop()
```

- 配列が空だった場合は undefined を返す。

## 再実装

```js
Array.prototype.pop = function() {
  const list = array => Array.from(Array(array.length))
  try {
    return list(this).reduce((acc, cur, index) => this[index], undefined)
  } finally {
    if (this.length) this.length -= 1
  }
}
```

実装自体は単純で、reduce をひたすら回して最後に返った `this[index]` が最後の要素だよね、という動きになっている。最後が疎の要素だった場合は、undefined を返す。

try/finally が若干危険な感じなんだけど…個人的には return した後に length を-1 するという意図がこの形だと読み取り易いと思っている。

### 7/25 修正

ほんのちょっと修正、list関数はいらんかった。

```js
Array.prototype.pop = function() {
  try {
    return Array.from(this).reduce((acc, cur, index) => this[index], undefined)
  } finally {
    if (this.length) this.length -= 1
  }
}
```

というかこれ以前のメソッドでも全部いらんかった。

---

<h1 style="font-size: 4em;">push</h1>

[Array.prototype.push() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/push)

pop と対になる（？）メソッドで、配列の末尾に要素を追加する。

## 構文

```js
array.push(element1, ..., elementN)
```

- 追加する要素は複数指定が可能。
- 戻り値として、要素を追加した後の配列長を返す。

## 再実装

```js
Array.prototype.push = function(...args) {
  return args.reduce((acc, cur) => {
    this[this.length] = cur
    return this.length
  }, 0)
}
```

引数をレスト構文で配列にし、reduce で各要素を元配列（`this`）に追加していく。

疎の要素を考慮しなくてよいので非常にシンプルだ。元の配列が疎な配列だったとしても、後ろに追加していくのだから関係ないしね！

# おわりに

疎な配列を考慮するかしないかで大幅にコードが変わってしまった。

例えば疎な配列を全く考慮しない場合、map は以下のように簡潔に書ける。

```js
Array.prototype.map = function(callback, thisArgs) {
  return this.reduce((acc, cur, index, array) => {
    acc[index] = callback.call(thisArgs, this[index], index, array)
    return acc
  }, [])
}
```

当初はこんな感じで「わー reduce なら何でもできちゃうんだー」ってやるつもりだったけど、疎な配列を回避するための関数が幅を利かせすぎていて、reduce が単なるルールの一つになってる気がする。

…まあいいか。

つづく

## 参考

疎な配列（`Array(1)` のように要素そのものが無い）を判定する方法を参考にさせて貰った。

- [JavaScript の配列のパターン | Web Scratch](https://efcl.info/2016/10/11/array-patterns/)
