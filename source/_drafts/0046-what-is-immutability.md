---
title: '[JavaScript] イミュータブルにしろ、とは'
date: 2018-10-10 13:14:02
tags:
  - javascript
---

「オブジェクトはイミュータブルにしろ」とはよく聞く＆言われることだが、改めて考えだしたら分からなくなってきた。

イミュータブルにすると何がうれしいのか？ミュータブルでは何故ダメなのか…？

なので JavaScript での例を踏まえて考えてみたい。

<!-- more -->

# 動作環境

Node.js v8.11.3 の REPL を使って動作確認する。

# イミュータブルにする目的

イミュータブル（不変）にする目的、それは副作用を排除することであろう。

例えば ESLint には[no-param-reassign](https://eslint.org/docs/rules/no-param-reassign)というルールがあり、これは以下のように副作用のある代入を警告する。

例えば以下のようなコードは警告されると例示されている。

```js
function foo(bar) {
  bar.prop = 'value';
}
```

ではどういった時に問題となるか。

```js
function foo(bar) {
  bar.prop = 'after';
}

const obj = { prop: 'before' };
foo(obj);

console.log(obj.prop);
// > 'after'
```

はい、こういう事です。`foo`関数を通すことで`obj.prop`が変化してしまった。

これは`foo`関数を利用する側からしたら意図しない（であろう）ところで値を書き換えられている。つまり**foo 関数には副作用がある**と言え、問題を引き起こしやすいコードと言えるだろう。

なので ESLint では警告になるわけだね。

## const ＝イミュータブル、ではない

ちょっと脱線

「const なら書き換えできないんじゃないの？」と思う人もいると思う（いるよね？）

const は「変数の再代入を禁止」するのであって、宣言した変数の不変性を保証するものではない。つまり以下のように obj 変数自体を書き換えることは抑制できるけど

```js
> const obj = { a: 1 }
undefined
> obj = { b: 2 }
TypeError: Assignment to constant variable.
```

次のようにプロパティを変更することは抑制できない。

```js
> const obj = { a: 1 }
undefined
> obj.a = 2
2
> obj
{ a: 2 }
```

もちろん const を宣言することで再代入は不可能になり可変性は縮小されるので、可能な限り const 宣言とすべきだとは思うが、かと言って＝イミュータブルではないことは覚えておきたい。

# どこまで不変ならば良いのか

JavaScript ではプリミティブ型（number,string,boolean 等）はイミュータブルだが、Object や Array 等のオブジェクトはミュータブルだ。基本的に以後は Object/Array を以下にイミュータブルに保つか、という点を考えていく。

http://hhsprings.pinoko.jp/site-hhs/2018/02/eslint-no-param-reassign/
https://www.pandanoir.info/entry/2015/10/18/191106


# immutable.js
https://www.pandanoir.info/entry/2015/10/18/191106
http://tech.connehito.com/entry/2016/09/07/091428

# 不変性
http://sakaba.cocolog-nifty.com/sakaba/2014/05/post-4f14.html
https://nekogata.hatenablog.com/entry/2013/06/15/013752
