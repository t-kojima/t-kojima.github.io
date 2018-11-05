---
title: '[JavaScript] Array()とnew Array()は何が違うのか'
date: 2018-11-03 09:54:48
tags:
  - javascript
---

`Array()`と`new Array()`は何が違うのだろうか？普通に使っている分にはどちらも同じように使えるが、毎度どちらがいいのか迷ってしまう。

大抵そういう場合は`Array()`を使うが、それでいいのだろうか？調べてみた。

<!-- more -->

# 実行環境

- Node 8.11.3

以下、Node の RPEL で動作確認する。

# 単純な動作比較

引数を指定して使用してみる。

```js
> Array(3)
[ <3 empty items> ]
> new Array(3)
[ <3 empty items> ]
```

両方とも`[ <3 empty items> ]`が返る。

ここが迷うポイント、どっちがいい？どっちでもいい？となってしまう。

# new の有無

結局は`new`を省略できるかどうかという話のようだ

[オブジェクト | JavaScript プログラミング解説](https://so-zou.jp/web-app/tech/programming/javascript/grammar/object/#omitted-new-operator)では new 演算子の有無による違いをわかりやすく表にしてある。ここでは Array は new の有無に差異はなく、どちらも Array オブジェクトを返すとある。

また、[new - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Operators/new)にも省略時の記述が以下のようにある。

> new 演算子を記述しない場合、コンストラクタは通常の関数として扱われ、オブジェクトを作成しません。その際、 this の値は異なります。

なるほど、new を省略するとコンストラクタが通常の関数として呼ばれるのか。
