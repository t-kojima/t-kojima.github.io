---
title: 'ネストの深いArrayをフラットな文字列にする'
date: 2018-06-07 14:50:30
tags:
- javascript
---

ネストの深い Array に flatten 的な動きをさせたい。

<!-- more -->

## やりたいこと

これを

```js
array = [1, 2, [3, 4, [5, 6, [7]]]]
```

こうしたい

```js
str = '1 2 3 4 5 6 7'
```

## やり方

`toString()`して`replace`する

```js
const array = [1, 2, [3, 4, [5, 6, [7]]]]
const str = array.toString().replace(/,/g, ' ')

// => '1 2 3 4 5 6 7'
```

## 疑似 flatten

文字列にしたものを`split`すると元の配列を`flatten`した感じになる。

疑似って言ってるのは数値が文字列になってしまったり、要素が文字列の時にカンマを含むと破綻するから

```js
const array = [1, 2, [3, 4, [5, 6, [7]]]]
const str = array.toString().replace(/,/g, ' ')
const flat = str.split(' ')

// => [ '1', '2', '3', '4', '5', '6', '7' ]
```

数値は数値のままにしておきたい場合

```js
const array = [1, 2, ['a', 'b', [5, 6, [7]]]]
const str = array.toString().replace(/,/g, ' ')
const flat = str.split(' ').map(a => Number(a) || a)

// => [ 1, 2, 'a', 'b', 5, 6, 7 ]
```

条件付きだけど、手軽に変換できて良いのではないかな

## どういう使い方をしたかったのか

[https://developer.cybozu.io/hc/ja/articles/202166310#step3](https://developer.cybozu.io/hc/ja/articles/202166310#step3)

上記ページの「返り値」の部分、この動きを実装したかった。

```js
// パラメータにオブジェクトや配列が渡された場合、クエリ文字列は以下のように展開されます。
// params = {foo: 'bar', record: {key: ['val1', 'val2']}}
foo=bar&record.key[0]=val1&record.key[1]=val2
```

こんな感じで実装

```js
const getQuery = obj =>
  Object.keys(obj).reduce((result, key) => {
    if (Array.isArray(obj[key])) {
      result.push(...obj[key].map((a, i) => `${key}[${i}]=${a}`))
    } else if (obj[key] instanceof Object) {
      result.push(getQuery(obj[key]).map(a => `${key}.${a}`))
    } else {
      result.push(`${key}=${obj[key].toString()}`)
    }
    return result
  }, [])

const query = getQuery(params)
  .toString()
  .replace(/,/g, '&')
```

`params`に

```js
{foo: 'bar', record: {key: ['val1', 'val2']}}
```

を投入すると、`query`には

```js
foo=bar&record.key[0]=val1&record.key[1]=val2
```

が入る。
