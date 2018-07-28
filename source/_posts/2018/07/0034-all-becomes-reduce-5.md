---
title: '全てがReduceになる(5)'
date: 2018-07-28 01:16:49
tags:
- javascript
- nodejs
---

[前回](https://t-kojima.github.io/2018/07/26/0033-all-becomes-reduce-4/)に続き、Array のメソッドを reduce で再実装する。

今回は Node における Array#sort のアルゴリズムの解明で非常に長い内容になってしまった。なので sort のみです。

- sort

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

<h1 style="font-size: 4em;">sort</h1>

[Array.prototype.sort() - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/sort)

sort なので当然配列の並び替えを行う。ただしこの sort メソッドは非常にクセがある。（後述）

## 構文

```js
arr.sort([compareFunction])
```

sort はソートされた配列を戻り値として返すが、レシーバとなる配列を破壊的にソートする。

compareFunction を指定した場合、その関数の結果に基づいたソートを行い、省略した場合は各要素を文字列比較した結果に基づき、辞書順次ソートする。

## 再実装

```js
const concat = (...args) =>
  args.reduce((acc, array) => {
    Array.from(array).reduce((_, cur, index) => {
      if (index in array) {
        acc[acc.length] = array[index]
      } else {
        acc.length += 1
      }
    }, null)
    return acc
  }, [])

const push = (from, to, index) => {
  if (index in from) {
    to[to.length] = from[index]
  } else {
    to.length += 1
  }
}

const quickSort = (array, compare) => {
  if (array.length < 1) return array
  // 後でconcatする為配列で保持
  const pivot = 0 in array ? [array[0]] : [,]
  const [left, right] = Array.from(array).reduce(
    (acc, cur, index) => {
      if (index !== 0) {
        // プロパティ有 & compareが0以下ならright
        const idx =
          0 in pivot && compare.call(undefined, pivot[0], array[index]) <= 0
        push(array, acc[Number(idx)], index)
      }
      return acc
    },
    [[], []] // [[left], [right]] を返す
  )
  return concat(quickSort(left, compare), pivot, quickSort(right, compare))
}

Array.prototype.sort = function(compare) {
  const sorted = quickSort(this, (x, y) => {
    if (x === undefined || y === undefined) {
      return (x === undefined ? 1 : 0) + (y === undefined ? -1 : 0)
    }
    if (compare) {
      const v = Number(compare.call(undefined, x, y))
      return Number.isNaN(v) ? 0 : v
    }
    return (String(x) > String(y) ? 1 : 0) + (String(x) < String(y) ? -1 : 0)
  })

  this.length = 0

  return Array.from(sorted).reduce((acc, cur, index) => {
    push(sorted, acc, index)
    return acc
  }, this)
}
```

まずいきなりだが、配列をソートする。

```js
const sorted = quickSort(this, (x, y) => {
  if (x === undefined || y === undefined) {
    return (x === undefined ? 1 : 0) + (y === undefined ? -1 : 0)
  }
  if (compare) {
    const v = Number(compare.call(undefined, x, y))
    return Number.isNaN(v) ? 0 : v
  }
  return (String(x) > String(y) ? 1 : 0) + (String(x) < String(y) ? -1 : 0)
})
```

ソートにはクイックソートを使う。
`quickSort` 関数は名前の通りクイックソートを行う関数で、第一引数にソート対象の配列、第二引数には callback を渡す。この時 quickSort に渡す callback は比較関数である compareFunction を渡すのだが、 sort メソッドの引数である compareFunction をそのまま渡すのではなく、 undefined の場合や compareFunction が未定義の場合の比較処理を追加した関数でラップしている。（仕様にある"文字列で比較し辞書順にソート"の部分である）

これは ECMAScript 仕様書の[Runtime Semantics: SortCompare(x,y)](https://www.ecma-international.org/ecma-262/6.0/#sec-sortcompare)にある手順をそのまま実装している。

クイックソート部分は[クイックソート : アルゴリズム](https://www.codereading.com/algo_and_ds/algo/quick_sort.html)を参考にさせて頂いて、以下のようになっている。

```js
const quickSort = (array, compare) => {
  if (array.length < 1) return array
  // 後でconcatする為配列で保持
  const pivot = 0 in array ? [array[0]] : [,]
  const [left, right] = Array.from(array).reduce(
    (acc, cur, index) => {
      if (index !== 0) {
        // プロパティ有 & compareが0以下ならright
        const idx =
          0 in pivot && compare.call(undefined, pivot[0], array[index]) <= 0
        push(array, acc[Number(idx)], index)
      }
      return acc
    },
    [[], []] // [[left], [right]] を返す
  )
  return concat(quickSort(left, compare), pivot, quickSort(right, compare))
}
```

配列の先頭要素を pivot として取り出し、他の要素と compare 関数で比較し、left と right に割り振っていく。その後 left と right 配列は再帰的にクイックソートを繰り返し、最終的に left+pivot+right で配列を結合する形になる。

concat は単純に配列を結合する関数、push は配列に要素を追加する関数だ。concat は配列同士しか結合できないようになっているので、pivot は配列で保持している。

また、疎な配列であってもソートできるようにしていて、pivot がプロパティの無い要素（疎要素）ならば、他の要素は無条件で left（先頭側）にソートするようになっている。これによりプロパティの無い要素（疎要素）は compare の結果にかかわらず全て配列の末尾側にまとめられる。

この仕様も[Runtime Semantics: SortCompare(x,y)](https://www.ecma-international.org/ecma-262/6.0/#sec-sortcompare)の NOTE1 に記載がある。

> Because non-existent property values always compare greater than undefined property values, and undefined always compares greater than any other value, undefined property values always sort to the end of the result, followed by non-existent property values.

## ネイティブの sort と結果が異なる問題

大問題なわけで、僕のソート実装が間違ってるに等しいわけで

具体的には以下の配列をに比較関数有りでソートした時に結果が異なる。（`※以降、node組み込みのsortをネイティブ、自作のsortをリメイクと呼ぶ`）

```js
const array = [1, undefined, 'z', 2, 'a', 0, null]
```

比較関数無しの場合はネイティブもリメイクも同じ結果を返す。

```js
> array = [1, undefined, 'z', 2, 'a', 0, null];
> array.sort();
[0, 1, 2, 'a', null, 'z', undefined]
```

しかし `(x, y) => x - y` という比較関数を与えると異なる結果を返してしまう。

```js
// ネイティブ
> array = [1, undefined, 'z', 2, 'a', 0, null];
> array.sort((x, y) => x - y);
[null, 1, 'z', 2, 'a', 0, undefined]

// リメイク
> array = [1, undefined, 'z', 2, 'a', 0, null];
> array.sort((x, y) => x - y);
[0, null, 1, 'z', 2, 'a', undefined]
```

上記の通り、0 の位置が異なる。

### Node.js(v8) のソースを見てみる。

[v8/array.js at master · v8/v8](https://github.com/v8/v8/blob/676501a8d7e208e16356027925b2ee60bd4fbc9c/src/js/array.js#L645)

Node.js は v8 を使用しているので、sort のコードを確認してみる。（場所は上記の箇所で良いハズ…）

いきなり大事なことが書いてある。ソートアルゴリズムはクイックソートを使用しているが、配列長が 10 以下の場合は挿入ソートを使用しているらしい。

> // In-place QuickSort algorithm.  
> // For short (length <= 10) arrays, insertion sort is used for efficiency.

その挿入ソートに対応するコードは以下のものだ

```js
function InsertionSort(a, from, to) {
  for (var i = from + 1; i < to; i++) {
    var element = a[i]
    for (var j = i - 1; j >= from; j--) {
      var tmp = a[j]
      var order = comparefn(tmp, element)
      if (order > 0) {
        a[j + 1] = tmp
      } else {
        break
      }
    }
    a[j + 1] = element
  }
}
```

このコード使って疑似的なネイティブのソートを再現し、どのような結果を返すか確認してみる。

```js
const compare = (x, y) => x - y

function comparefn(x, y) {
  if (x === undefined || y === undefined) {
    return (x === undefined ? 1 : 0) + (y === undefined ? -1 : 0)
  }
  if (compare) {
    const v = Number(compare.call(undefined, x, y))
    return Number.isNaN(v) ? 0 : v
  }
  return (String(x) > String(y) ? 1 : 0) + (String(x) < String(y) ? -1 : 0)
}

function InsertionSort(a, from, to) {
  for (var i = from + 1; i < to; i++) {
    var element = a[i]
    for (var j = i - 1; j >= from; j--) {
      var tmp = a[j]
      var order = comparefn(tmp, element)
      if (order > 0) {
        a[j + 1] = tmp
      } else {
        break
      }
    }
    a[j + 1] = element
  }
}

array = [1, undefined, 'z', 2, 'a', 0, null]
InsertionSort(array, 0, array.length)
console.log(array)
```

結果、ネイティブと違うソート結果になってしまった。。。

```js
// 元配列
> [1, undefined, 'z', 2, 'a', 0, null]

// ネイティブ
> [null, 1, 'z', 2, 'a', 0, undefined]
// リメイク
> [0, null, 1, 'z', 2, 'a', undefined]
// 疑似ネイティブ
> [1, 'z', 2, 'a', 0, null, undefined]

// 全部ちがう。。。
```

更に詳細な動作を確認するため、InsertionSort の `a[j + 1] = element` 箇所で console.log を仕込み、配列の途中経過を出力するようにしてみた。

```bash
[ 1, undefined, 'z', 2, 'a', 0, null ]
[ 1, 'z', undefined, 2, 'a', 0, null ]
[ 1, 'z', 2, undefined, 'a', 0, null ]
[ 1, 'z', 2, 'a', undefined, 0, null ]
[ 1, 'z', 2, 'a', 0, undefined, null ]
[ 1, 'z', 2, 'a', 0, null, undefined ]
[ 1, 'z', 2, 'a', 0, null, undefined ]
```

すると undefined がただ後ろに移動していくだけの動きをしていることが分かった。

しかし[挿入ソートのアルゴリズム](https://www.codereading.com/algo_and_ds/algo/insertion_sort.html)を改めて確認すると、`1, z, 2, a, 0` の並びは 数値と文字を比較すると結果が全て 0 になるため、隣同士の比較だと動くことができなくて、さらに 0 と null の比較も結果は 0 になる（null は計算上 0 扱いの為）ので、0 と null の比較も動かない。結局 undefined だけが 0 以外の結果を返し、移動していく形になっている。つまり動作としては何ら問題がないことになる。

ネイティブのようにnullを先頭にやらないといけないのだが、これどうやっても null は先頭にいかないよな…？

### 改めて v8 のコードを読む

改めて v8 のコードを読んでいると、[大事なコメント](https://github.com/v8/v8/blob/676501a8d7e208e16356027925b2ee60bd4fbc9c/src/js/array.js#L792)を見逃していた。

> We also move all non-undefined elements to the front of the array and move the undefineds after that. Holes are removed.

（意訳）non-undefined な要素を配列の先頭に移動し、その後ろに undefined な要素を移動する。そうすると穴が消える

この処理は `%PrepareElementsForSort(array, length);` というC++の関数で行われているようだ。ここから処理を辿っていき、動作を詳しく見てみる。

- [v8/runtime-array.cc at v8/v8#L356](https://github.com/v8/v8/blob/676501a8d7e208e16356027925b2ee60bd4fbc9c/src/runtime/runtime-array.cc#L356)
- [v8/runtime-array.cc at v8/v8#L157](https://github.com/v8/v8/blob/676501a8d7e208e16356027925b2ee60bd4fbc9c/src/runtime/runtime-array.cc#L157)

すると配列の操作をしていると思われる場所が…あった

```cpp
    unsigned int undefs = limit;
    unsigned int holes = limit;
    // Assume most arrays contain no holes and undefined values, so minimize the
    // number of stores of non-undefined, non-the-hole values.
    for (unsigned int i = 0; i < undefs; i++) {
      Object* current = elements->get(i);
      if (current->IsTheHole(isolate)) {
        holes--;
        undefs--;
      } else if (current->IsUndefined(isolate)) {
        undefs--;
      } else {
        continue;
      }
      // Position i needs to be filled.
      while (undefs > i) {
        current = elements->get(undefs);
        if (current->IsTheHole(isolate)) {
          holes--;
          undefs--;
        } else if (current->IsUndefined(isolate)) {
          undefs--;
        } else {
          elements->set(i, current, write_barrier);
          break;
        }
      }
}
```

前述のコメント通り、undefined と hole(疎要素)を配列の後方へ移動させる処理だ。その内容はざっくりと言うと、**配列中に undefined か hole(疎要素)が見つかったら、最後尾の要素とスワップする**というものだ。

なので `%PrepareElementsForSort(array, length);` が実行された結果、array は以下の並びになっている（はずだ）。

```js
;[1, null, 'z', 2, 'a', 0, undefined]
```

改めてこの配列で擬似ネイティブのソートを実行すると、こんどはネイティブの結果と同等になった。

```js
array = [1, null, 'z', 2, 'a', 0, undefined]
InsertionSort(array, 0, array.length)
console.log(array)
// > [null, 1, 'z', 2, 'a', 0, undefined ]
```

つまりリメイクの sort メソッドは以下の点を改良するとネイティブと同じ結果を返すようになる。

- undefined と hole（疎要素）の移動はソート処理前に行い、最後尾の要素とのスワップで行う。
- 配列長が 10 以下の場合、クイックソートではなく挿入ソートで実行する。

# で、直すの？

じつはリメイクのコードでも undefined と hole（疎要素）の移動は実現できている。ただ移動のアルゴリズムでスワップを行っていないだけだ。そしてこの移動アルゴリズムとソートアルゴリズムは ECMAScript の仕様書では規定されておらず、実装依存となっている。

今回リメイクとネイティブの差異が何処にあるかという点で深堀りしてみたが、リメイクのコードは現状でも仕様を満たしているので、修正はせずにこれで完成としたい。（というかこの調査でもう疲れきった…）

Array のメソッドはまだ少しあるので、そちらに力を割くことにしよう…

つづく

## （余談）ブラウザで実行してみたら…

JSFiddle で以下のコードを実行してみたところ、Firefox のみ違うソート結果となってしまった。node と Chrome は同じ v8 を使っているから同じ結果なのだろうか。Array#sort が頻繁に実装依存と言われているのはこういうことなんだな。

### node

```js
array = [1, undefined, 'z', 2, 'a', 0, null]
array.sort((x, y) => x - y)
console.log(array)

// > [ null, 1, 'z', 2, 'a', 0, undefined ]
```

### firefox

```js
array = [1, undefined, 'z', 2, 'a', 0, null]
array.sort((x, y) => x - y)
console.log(array)

// > [0, null, 'a', 'z', 1, 2, undefined]
```

## Chrome

```js
array = [1, undefined, 'z', 2, 'a', 0, null]
array.sort((x, y) => x - y)
console.log(array)
// > [null, 1, "z", 2, "a", 0, undefined]
```

# 参考

- [言語仕様から入る JavaScript 入門 その 1 - Panda Noir](https://www.pandanoir.info/entry/2016/04/16/190000)
- [ECMAScript 2015 Language Specification – ECMA-262 6th Edition](https://www.ecma-international.org/ecma-262/6.0/#sec-array.prototype.sort)
- [v8/v8: The official mirror of the V8 Git repository](https://github.com/v8/v8)

<!-- ### 謎な部分がある

InsertionSort を実行する前、InnerArraySort を呼び出した直後に以下のブロックがある。`!IS_CALLABLE(comparefn)` となっていることから、comparefn が未定義の場合に comparefn のデフォルト動作を定義する箇所に思えるが、その定義される内容が[Runtime Semantics: SortCompare(x,y)](https://www.ecma-international.org/ecma-262/6.0/#sec-sortcompare)と若干異なるのだ。そもそも仕様上は comparefn の定義有無に関わらずこの処理はあるべき（Step4 で comparefn が呼ばれる為）なのだが、下記のロジックからは Step4 が抜けている。

以下のコードが正しいのであれば、comparefn が定義されている場合、仕様通りの動きをしないのではないか？？

```js
  if (!IS_CALLABLE(comparefn)) {
    comparefn = function (x, y) {
      if (x === y) return 0;
      if (%_IsSmi(x) && %_IsSmi(y)) {
        return %SmiLexicographicCompare(x, y);
      }
      x = TO_STRING(x);
      y = TO_STRING(y);
      if (x == y) return 0;
      else return x < y ? -1 : 1;
    };
}
``` -->

<!-- ## 22.1.3.24 Array.prototype.sort (comparefn)

`このセクションはECMAScript仕様書の和訳（自力）です。`

配列の要素をソートします。ソートは必ずしも安定はしません。（つまり、比較される要素は必ずしも最初の並び順にあるわけではありません。）comparefn が定義されている場合、それは x と y 二つの引数を持つ関数である必要があり、x < y の場合は -1、x = y の場合は 0、x > y の場合は 1 を返します。

最初に、ソート機能を初期化する為に以下のステップが実行されます。

```text
1. obj に ToObject(this) が代入される。         Let obj be ToObject(this value).
2. len に ToLength(obj.length) が代入される。   Let len be ToLength(Get(obj, "length")).
3. ReturnIfAbrupt(len) を実行する。             ReturnIfAbrupt(len).
```

ソートメソッドの仕様では、オブジェクト(obj)が下のアルゴリズムで true を返す時、疎であると言える。

```text
1. 数値 i を 0 <= i < len の範囲で列挙する。                     For each integer i in the range 0≤i< len
    a. elem に obj.GetOwnProperty(ToString(i)) を代入する。     Let elem be obj.[[GetOwnProperty]](ToString(i)).
    b. もし elem が未定義(undefined)の場合、trueを返す。         If elem is undefined, return true.
2. falseを返す。                                               Return false.
```

ソートの順序は昇順で、このファンクションが完了した後に、整数インデックスが len 以下の obj のプロパティ値で並び替えされる。ソート機能の結果は次のように決定される。

comparefn が定義済みかつ、この配列の要素に対し一貫性のある比較関数ではない場合（下記参照）、**ソート順は実装定義です**。comparefn が未定義で、SortCompare（22.1.3.24.1）が一貫性のある比較関数として動作しない時も、**ソート順は実装定義です**。

proto に obj.GetPrototypeOf()を代入する。もし proto が null ではなく、以下の条件を全て満たす整数 j が存在する場合、**ソート順は実装定義**です。

```text
- obj は疎                                  obj is sparse
- 0 <= j < len                              0 ≤ j < len
- HasProperty(proto, ToString(j))は true    HasProperty(proto, ToString(j)) is true.
```

obj が疎で以下のいずれかの条件に合致するなら**ソート順は実装定義**です。

```text
- IsExtensible(obj) is false.
- 正の整数インデックスがlen以下のいずれかのプロパティで、Configurable属性がfalseのデータプロパティの場合
        Any integer index property of obj whose name is a nonnegative integer less than len is a data property whose [[Configurable]] attribute is false.
```

obj が以下のいずれかの条件に合致するなら**ソート順は実装定義**です。

```text
- objが（プロキシエキゾチックオブジェクトを含む）エキゾチックオブジェクトで、Get、Set、Delete、GetOwnPropertyの動作はこれらの内部メソッドの通常のオブジェクト実装では無い場合
    If obj is an exotic object (including Proxy exotic objects) whose behaviour for [[Get]], [[Set]], [[Delete]], and [[GetOwnProperty]] is not the ordinary object implementation of these internal methods.

- 正の整数インデックスがlen以下のいずれかのプロパティでアクセサプロパティか、Writable属性がfalseのデータプロパティの場合
    If any index index property of obj whose name is a nonnegative integer less than len is an accessor property or is a data property whose [[Writable]] attribute is false.

- comparefnが未定義で、SortCompareに変更されたobjかいくつかのオブジェクトのプロトタイプチェーンにToStringを適用した値を引数として渡した場合
    If comparefn is undefined and the application of ToString to any value passed as an argument to SortCompare modifies obj or any object on obj’s prototype chain.

- comparefnが未定義かつ全てにToStringを適用した、SortCompareにいくつかの特定の値を引数として渡した
    If comparefn is undefined and all applications of ToString, to any specific value passed as an argument to SortCompare, do not produce the same result.
``` -->
