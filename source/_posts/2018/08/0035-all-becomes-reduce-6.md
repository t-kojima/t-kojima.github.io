---
title: '全てがReduceになる(6)'
date: 2018-08-01 20:40:02
tags:
- javascript
- nodejs
---

[前回](https://t-kojima.github.io/2018/07/28/0034-all-becomes-reduce-5/)に続き、残りのメソッドをやっつける。消化試合とも言う。

- splice
- toLocaleString
- toString
- unshict

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

<h1 style="font-size: 4em;">splice</h1>

[Array.prototype.splice() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/splice)

指定した範囲の要素を取り除く（戻り値として返す）したり、取り除いた場所に要素を追加したりできるメソッド。若干機能過多（？）な気がしないでもない

## 構文

```js
array.splice(index, howMany, [element1][, ..., elementN]);
array.splice(index, [howMany, [element1][, ..., elementN]]);  // SpiderMonkey/Firefox 拡張、この場合 howMany=array.length-index となります
```

引数も多く若干複雑な挙動を取るメソッドなので、各引数に関わる仕様を以下に列挙する。

- 戻り値として取り除かれた要素の配列を返す。

### index

- 要素を切り取る開始位置を指定する。
- index を省略すると何も処理を行わない。
- 数値への暗黙的な変換ができない要素を指定した場合、0 として扱われる。
- 負の値を指定した場合、配列の末尾からのオフセット値となる。
- 負の値を指定した場合、値の絶対値が配列長を超える場合、0 として扱われる。

### howMany

- 切り取る要素の数を指定する。
- howMany を省略した場合、length - index として扱われる。
- 数値への暗黙的な変換ができない要素を指定した場合、0 として扱われる。
- length - index を超える値を指定した時、length-index として扱われる。
- 0 以下の値を指定した時、0 として扱われる。

### element1...elementN

- 追加する要素を指定する。複数指定可能。

## 再実装

上記の仕様を再実装すると以下のようになる。

```js
/* eslint-disable array-callback-return */
Array.prototype.splice = function(idx = this.length, howMany, ...elements) {
  const i =
    Math.min(
      Math.max(Number.parseInt(idx, 10) + (idx < 0 ? this.length : 0), 0),
      this.length
    ) || 0
  const h =
    Math.min(
      Math.max(Number.parseInt(howMany || this.length - i, 10), 0),
      this.length - i
    ) || 0

  // 配列のi位置にelementsを挿入した配列コピーを返す
  const copyWithElements = Array.from(
    Array(this.length + elements.length)
  ).reduce((acc, cur, index) => {
    if (index < i && index in this) {
      acc[index] = this[index]
    }
    if (index >= i && index < i + elements.length) {
      acc[index] = elements[index - i]
    }
    if (index >= i + elements.length && index - elements.length in this) {
      acc[index] = this[index - elements.length]
    }
    return acc
  }, Array(this.length + elements.length))

  this.length = 0
  this.length = copyWithElements.length - h

  // index - i - elements.lengthからh個の要素を切り出す。
  // (事前にelementsを挿入しているので、elements.length分ズレる)
  return copyWithElements.reduce((acc, cur, index) => {
    if (index < i + elements.length) {
      this[index] = cur
    }
    if (index >= i + elements.length && index < i + h + elements.length) {
      acc[index - i - elements.length] = cur
    }
    if (index >= i + h + elements.length) {
      this[index - h] = cur
    }
    return acc
  }, Array(h))
}
```

いくつかの処理に分かれているので、各々の処理を詳細に見ていく。

```js
const i =
  Math.min(
    Math.max(Number.parseInt(idx, 10) + (idx < 0 ? this.length : 0), 0),
    this.length
  ) || 0
const h =
  Math.min(
    Math.max(Number.parseInt(howMany || this.length - i, 10), 0),
    this.length - i
  ) || 0
```

最初に index,howMany に対し、前述の仕様に沿った変換を掛ける。howMany は index と違い、マイナス値を許さず、最大値が length-index となる。

続いて、copyWithElements という配列に追加要素を含んだ元配列をコピーする。

```js
const copyWithElements = Array.from(
  Array(this.length + elements.length)
).reduce((acc, cur, index) => {
  if (index < i && index in this) {
    acc[index] = this[index]
  }
  if (index >= i && index < i + elements.length) {
    acc[index] = elements[index - i]
  }
  if (index >= i + elements.length && index - elements.length in this) {
    acc[index] = this[index - elements.length]
  }
  return acc
}, Array(this.length + elements.length))
```

例えば元配列が `[1,2,3]` で、追加要素が `'a', 'b'`、index が `1` の場合、copyWithElements は `[1,'a','b',2,3]` になる。

本来一回の reduce で要素の削除と追加を同時に行いたかったが、追加要素数と削除要素数が必ずしも一致しないことから処理が難しく（必要以上に複雑になる）、追加と削除は別のロジックとしたほうが理解しやすいと感じた。

```js
this.length = 0
this.length = copyWithElements.length - h
```

元配列を書き換えるメソッドなので、`lentgh = 0` で一度配列をリセットし、最終的な配列長をセットする。

最後に index 位置から howMany 分の要素を切り出す。

```js
// index - i - elements.lengthからh個の要素を切り出す。
// (事前にelementsを挿入しているので、elements.length分ズレる)
return copyWithElements.reduce((acc, cur, index) => {
  if (index < i + elements.length) {
    this[index] = cur
  }
  if (index >= i + elements.length && index < i + h + elements.length) {
    acc[index - i - elements.length] = cur
  }
  if (index >= i + h + elements.length) {
    this[index - h] = cur
  }
  return acc
}, Array(h))
```

コメントにもあるが、先に要素を追加している状態なので、切り出す index の位置は element.length 分ずれることになる。

かなり複雑になってしまったが、これで splice 再実装できた。

---

<h1 style="font-size: 4em;">toLocaleString</h1>

[Array.prototype.toLocaleString() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/toLocaleString)

配列の各要素に `toLocaleString` メソッドを適用する。

## 構文

```js
arr.toLocaleString()
```

引数が無い。シンプル！

## 再実装

再実装コードは以下の通りだ。

```js
Array.prototype.toLocaleString = function() {
  const value = v => (v === undefined || v === null ? '' : v)

  // 空配列は空文字列を返す
  if (this.length === 0) return ''

  const copy = Array.from(this).reduce((acc, cur, index) => {
    acc[index] = value(cur).toLocaleString()
    return acc
  }, [])

  return copy.reduce((acc, cur) => `${acc},${cur}`)
}
```

まず join と同様に、undefined,null,プロパティ無しの要素は全て空白として扱われる。value 関数はその変換を行う。

次に空配列は空文字列を返す。空配列の場合 reduce が一切回らないので固定の処理で良い。

次に配列のコピーを作成しつつ、各要素に value 関数と toLocaleString メソッドの適用を行う。

これは最後の連結処理と同時に以下の形でいけるかと思ったが、要素が 1 個の時 reduce が一切回らずにその要素をただ返す動作をした為、このように分割している。

```js
return Array.from(this).reduce(
  (acc, cur) => `${value(acc).toLocaleString()},${value(cur).toLocaleString()}`
)
```

reduce の initialValue を省略すると index が 1 から始まるが、この時 length が 1 だと index=1 の要素が存在しないので、reduce 自身が回らないのだろう。

---

<h1 style="font-size: 4em;">toString</h1>

[Array.prototype.toString() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/toString)

toLocaleString と同様で、配列の各要素に `toString` メソッドを適用するのみだ。

## 構文

```js
string = array.toString()
```

あまり気にしなくて良いと思うが、`Object.prototype.toString` を上書きしているらしい。

## 再実装

ハッキリ言って toLocaleString メソッドと全く一緒、以上！

```js
Array.prototype.toString = function() {
  const value = v => (v === undefined || v === null ? '' : v)

  // 空配列は空文字列を返す
  if (this.length === 0) return ''

  const copy = Array.from(this).reduce((acc, cur, index) => {
    acc[index] = value(cur).toString()
    return acc
  }, [])

  return copy.reduce((acc, cur) => `${acc},${cur}`)
}
```

---

<h1 style="font-size: 4em;">unshift</h1>

[Array.prototype.unshift() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/unshift)

やっとこ最後のメソッドだ。

unshift は配列の最初に要素を追加する。push の逆版だ。

## 構文

```js
arr.unshift(element1[, ...[, elementN]])
```

push と同様に element は複数指定が可能、そして戻り値として要素が追加された後の配列長を返す。

## 再実装

```js
Array.prototype.unshift = function(...elements) {
  const copy = this.reduce((acc, cur, index) => {
    acc[index] = cur
    return acc
  }, Array(this.length))

  this.length = 0
  this.length = copy.length + elements.length

  return copy.reduce(
    (acc, cur, index) => {
      this[index + elements.length] = copy[index]
      return acc
    },
    elements.reduce((acc, cur, index) => {
      this[index] = cur
      return acc
    }, copy.length + elements.length)
  )
}
```

元配列を copy する処理や length=0 などは何度も出てきた処理なので割愛

最後に return する箇所で reduce を二重に行っている。最初の reduce は追加要素（elements）をレシーバとして行っていて、その結果の配列を initialValue として次の reduce を行っている。そういう感じで追加要素を配列の最初に押し込んでいるのだ。

# おわりに

軽い気持ちでやり始めた反面、想像以上に大変だったけど、いい暇つぶしになった。今回扱ったメソッド仕様の理解は深まったので、個人的に得るものはあったと思う。（特に疎な配列と sort 関係は普通に使ってたらここまで調べないので）

つづかない
