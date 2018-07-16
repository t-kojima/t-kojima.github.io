---
title: '[Node] Mochaで非同期処理の例外ハンドリング'
date: 2018-07-16 11:09:53
tags:
- nodejs
- mocha
---

非同期処理（Promise とか、最近では async/await とか）内で発生した例外を mocha でテストする場合、いくつかの方法があるのでまとめてみる。

<!-- more -->

# おさらい

その前に少しおさらい。

同期処理での例外のテストと、例外をスローしない非同期処理のテストを見てみる。

## 同期処理での例外テスト

chai の expect を使って例外のスローを判定する。expect には `関数` を渡す必要があるので、`expect(expectedFunction)`という形になっている。

```js
const { expect } = require('chai')

describe('同期処理テスト', () => {
  it('例外がスローされること', () => {
    expect(expectedFunction).to.throw()
  })
})
```

検査対象の関数に引数がある場合は、以下のように無名関数でラップしてあげればよい。

```js
expect(() => expectedFunction(args)).to.throw()
```

## 例外をスローしない非同期処理のテスト

単純に非同期関数をテストする場合だ、mocha は async/await が使えるので非常にシンプルに書くことができる。

```js
const { assert } = require('chai')

describe('非同期処理テスト', () => {
  it('actualがexpectedであること', async () => {
    const actual = await expectedFunction()
    assert.equal(actual, expected)
  })
})
```

done を使っても書けるが、若干冗長になる。

```js
const { assert } = require('chai')

describe('非同期処理テスト', () => {
  it('actualがexpectedであること', done => {
    expectedFunction()
      .then(actual => {
        assert.equal(actual, expected)
        done()
      })
      .catch(err => {
        assert.fail()
        done()
      })
  })
})
```

mocha は Promise を return すると、mocha はちゃんと Promise のテストであると判断して処理してくれる。これを利用するともう少し簡素に記述できる。

```js
const { assert } = require('chai')

describe('非同期処理テスト', () => {
  it('actualがexpectedであること', () => {
    return expectedFunction().then(actual => {
      assert.equal(actual, expected)
    })
  })
})
```

おさらいお終い。

# 非同期処理の例外ハンドリング

さて本題だ、以下の関数のテストを行いたい。

```js
const fs = require('fs')
const { promisify } = require('util')

const getContent = path => {
  return promisify(fs.readFile)(path)
}
```

fs.readFile を promisify した関数で、ファイルの内容（content）を Promise で返す。こんな感じで呼び出すことができる。

```js
const content = await getContent('filepath')
console.log(content)
```

もしくは

```js
getContent('filepath').then(content => console.log(content))
```

## 単純に async/await を付けてみる

it のコールバックに async、expect に await を付けてみる。尚エラーを発生させるので、`missing.txt`という存在しないファイルを指定する。

```js
it('throw error', async () => {
  await expect(() => {
    getContent('./missing.txt')
  }).to.throw()
})
```

結果、怒られた

```bash
    1) throw error
(node:19780) UnhandledPromiseRejectionWarning: Error: ENOENT: no such file or directory, open './missing.txt'
(node:19780) UnhandledPromiseRejectionWarning: Unhandled promise rejection. This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch(). (rejection id: 2)
(node:19780) [DEP0018] DeprecationWarning: Unhandled promise rejections are deprecated. In the future, promise rejections that are not handled will terminate the Node.js process with a non-zero exit code.


  0 passing (28ms)
  1 failing

  1) file load
       throw error:
     AssertionError: expected [Function] to throw an error
```

Promise の Reject をハンドルしろ、とのことだ

## try/catch してみる

expect を介してだと async/await がうまく動かなかった。今度は expect を使わず、try/catch で判定してみる。

```js
it('throw error', async () => {
  try {
    await getContent('./missing.txt')
    assert.fail()
  } catch (e) {
    assert.equal(e.code, 'ENOENT')
  }
})
```

これはパスする。

```bash
  file load
    √ throw error


  1 passing
```

しかしあまり上手く無い感がある。`assert.fail()` とか `e.code` で判定している所とか。catch した Error オブジェクトを assert するのではなくて、できれば例外ハンドル自体を assert したい。

言い換えると例外がスローされたことを chai が検知して、assert が OK であると判断して貰いたい。上記の例だと、例外の検知はテストコードで行っているのでちょっとイケてないよね。ということだ。

## Promise を return する

おさらいの最後に書いた、Promise を return する形で書くと若干簡素になる。

```js
it('throw error', () =>
  getContent('./missing.txt').then(
    () => assert.fail(),
    err => assert.equal(err.code, 'ENOENT')
  ))
```

ただ前述の問題は解決していない。やっぱり例外ハンドル自体を assert したい。

## chai-as-promised を使う

`chai-as-promised` という chai のプラグインがあり、これを使うと Promise の判定がより直感的に記述できるようになる。

<a href="https://github.com/domenic/chai-as-promised" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

こんなに簡単になった。

```js
const chai = require('chai')
const chaiAsPromised = require('chai-as-promised')
chai.use(chaiAsPromised)

it('throw error', () => expect(getContent('./missing.txt')).to.be.rejected)
```

これなら例外ハンドルが飛んだことを `be.rejected` で判断してくれるので、大満足である。

エラーの型まで見たい場合は `be.rejectedWith()`を使えばいい

```js
const chai = require('chai')
const chaiAsPromised = require('chai-as-promised')
chai.use(chaiAsPromised)

it('throw error', () =>
  expect(getContent('./missing.txt')).to.be.rejectedWith(Error))
```

# 参考

- [mocha による非同期のテストの書き方 - 30 歳からのプログラミング](https://numb86-tech.hatenablog.com/entry/2017/04/02/212259)
- [Mocha が Promises のテストをサポートしました | Web Scratch](https://efcl.info/2014/0314/res3708/)
- [How to test an ES7 async function · Issue #415 · chaijs/chai](https://github.com/chaijs/chai/issues/415)
- [javascript - Testing rejected promise in Mocha/Chai - Stack Overflow](https://stackoverflow.com/questions/31845329/testing-rejected-promise-in-mocha-chai/37222388)
- [Chai - plugins/chai-as-promised](http://www.chaijs.com/plugins/chai-as-promised/)
