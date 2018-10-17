---
title: '[JavaScript] Arrayのメソッドはまだまだ貧弱問題'
date: 2018-10-17 10:43:31
tags:
  - javascript
---

JavaScript で配列を操作する機会は多いけど、目的にドンピシャなメソッドが無いことが多々ある。

今回は以下のことがやりたい。

- インデックスを指定して要素を追加
- インデックスを指定して要素を削除
- インデックスを指定して要素の書き換え

<!-- more -->

# Insert/Delete ないの？

例えば Ruby だと[Array#insert](https://docs.ruby-lang.org/ja/latest/method/Array/i/insert.html)や[Array#delete_at](https://docs.ruby-lang.org/ja/latest/method/Array/i/delete_at.html)のようなメソッドが今回は欲しい。（これも破壊的なメソッドなのでドンピシャではないけど）

しかし JavaScript の Array に insert/delete はなく、「インデックスを指定して要素を追加」できそうなメソッドは`Array#splice`しかなさそうだ。

## Array#splice ではダメなのか

「要素の追加」と「要素の削除（取り出し）」を一度にやろうとしているメソッドなので、あまり好きではない。

一見して何をしているのか分かり辛いのもポイント低い、これ瞬時にどうなるか分かります？

```js
const array = [0, 1, 2, 3, 4];
const p = array.splice(1, 2, 0);
```

正解は以下のようになる。…分かり辛い。

```js
array => [0, 0, 3, 4];
p => [1, 2];
```

ちなみに「インデックスを指定して要素を追加」と「インデックスを指定して要素を削除」ならこうなる。

```js
// insert
array.splice(index, 0, 'value');
// delete
array.splice(index, 1);
```

慣れなのかもしれないけど、上が insert で下が delete だと瞬時に判断は難しいと感じる。

そもそも破壊的なメソッドなので、あまり良くは無いです。はい。

## lodash なら…？

ネイティブ Array にないなら lodash だ！

と、思ったけど lodash にも insert/delete 相当の関数は無いみたいだ。[過去に機能要求はあったよう](https://github.com/lodash/lodash/issues/89)だが、reject されている。

しかし後半に`Array#slice`を利用した代替案が提示されている。

```js
const newArray = [...array.slice(0, index), 'value', ...arr.slice(index)];
```

`Array#splice`より分かりやすいか？と言えば微妙な感じしかないが、非破壊的な insert/delete としてはこれくらいしか無いのかなとも思う。

ちなみに以下のようになる。

```js
// insert
const insertedArray = [
  ...array.slice(0, index),
  'value',
  ...array.slice(index),
];
// delete
const deletedArray = [...array.slice(0, index), ...array.slice(index + 1)];
```

# インデックスを指定して要素の書き換え

`[0, 1, 2, 3, 4]`に対して、`2`を`hoge`に書き換えたい…としよう。

単純に考えると以下のようになる。

```js
const array = [0, 1, 2, 3, 4];
array[2] = 'hoge';
```

しかし要素を書き換えてしまっているので、イミュータブルにやってみよう。

```js
const array = [0, 1, 2, 3, 4];
// update
const updatedArray = [
  ...array.slice(0, index),
  'hoge',
  ...array.slice(index + 1),
];
```

前述の insert/delete と仕組みは同じだ。やっぱり冗長な感じはするけど…もっといい方法ないだろうか。

# まとめ

思っていたよりもスッキリ書けなかったので若干モヤモヤが残るが、一応目的は達成できた。言語ごとの思想の違いではあると思うけど、もっと便利なメソッドがネイティブに追加されればいいなと思う。

あと「イミュータブルにする」ということについて、有意性は理解しているがそれを上手く言語化できないことに気付いたので、別途エントリにしたいなと思う。

# 環境

- Windows 10
- Node v8.11.3
