---
title: '[Node] 高階関数内での非同期処理（async/await）をどう書くか'
date: 2018-07-18 08:05:15
tags:
- nodejs
---

JavaScript の Array には `forEach` や `map` や `filter` などの便利な高階関数が用意されている。これらは非常に便利なのだが、引数の関数（callback）内で非同期処理を行おうとすると途端に複雑になる。これらの高階関数内での非同期処理の利用について動作を確認したい。

<!-- more -->

# 準備

動作確認の準備として非同期の関数 `echo` を用意する。

```js
const echo = value => new Promise(resolve => setTimeout(resolve, 1000, value))
```

引数の値を 1 秒後に返すという関数で、以下のような動作をする。

```js
echo('test').then(console.log)

// 1. 1000ms待機
// 2. 'test'と出力
```

また、高階関数のレシーバとして配列を作る。

```js
const array = [0, 10, 20, 30, 40, 50, 60]
```

# forEach

まずはそのまま回して上手く動かないことを確認する。

```js
array.forEach(i => echo(i).then(console.log))
```

1 秒待機して全部一気に表示された

```bash
0
10
20
30
40
50
60
```

## forEach で async/await

async/await しても結果は同じだった。

```js
array.forEach(async i => {
  const ret = await echo(i)
  console.log(ret)
})
```

[async/await を、Array.prototype.forEach で使う際の注意点、という話 - Qiita](https://qiita.com/frameair/items/e7645066075666a13063) によると、forEach の `callback.call` に await が付いていないからそもそもダメで、for を使いなさいとのこと。

## forEach に reduce を使う

変則的な方法として、forEach の代替として reduce を使う。

```js
array.reduce(async (promise, i) => {
  await promise
  const ret = await echo(i)
  console.log(ret)
}, Promise.resolve())
```

これだと目論見通り、1 秒毎に console.log が実行される。

また、async/await を排除した以下の書き方でも成功する。

```js
array.reduce((promise, i) => {
  return promise.then(() => echo(i).then(console.log))
}, Promise.resolve())
```

## forEach の代わりに for...of を使う

安牌の解決方法、変に reduce とか使うより素直で分かりやすい。

```js
;(async () => {
  for (const i of array) {
    await echo(i).then(console.log)
  }
})()
```

# map

map では全部の要素を 2 倍して出力してみる。特に意味は無いけど…

例によってそのまま動かすと正しい結果が得られず、全て NaN になってしまう。そして表示してから 1 秒待機してしまう。

```js
console.log(array.map(item => echo(item) * 2))
// > [ NaN, NaN, NaN, NaN, NaN, NaN, NaN ]
// > 1000ms待機
```

2 倍しない場合は `Promise { <pending> }` の配列が返ってくるので、これを 2 倍しようとして NaN になってしまうのだろう。

## map で async/await

map の callback を async/await で装飾するだけだとやはり `Promise { <pending> }` の配列が返ってきてしまう。

```js
console.log(array.map(async i => (await echo(i)) * 2))
```

## Promise.all で結果を待つ

上記に加え、全体を Promise.all で囲うことで正しい結果を得ることができた。map の各要素は並列で処理されるので、待機時間も全体で 1 秒掛かり問題ない。

```js
Promise.all(array.map(async i => (await echo(i)) * 2)).then(console.log)
// > 1000ms待機
// > [ 0, 20, 40, 60, 80, 100, 120 ]
```

async/await 使わない版もいける

```js
Promise.all(array.map(i => echo(i).then(v => v * 2))).then(console.log)
```

## map の代わりに for...of を使う

結果自体は正しく取得できるが、シーケンシャルにループしてしまうので push が 1 回呼ばれる度に 1 秒掛かってしまう。

```js
;(async () => {
  const result = []
  for (const i of array) {
    result.push((await echo(i)) * 2)
  }
  console.log(result)
})()
```

これに関しては Promise.all で map を使ったほうがいいだろう。

# filter

filter では 20 の倍数だけの配列に変換するようにしてみよう。

素の状態ではやはり正しく結果は返ってこない。

```js
console.log(array.filter(i => echo(i) % 20 === 0))
// > []
// > 1000ms待機
```

map と同様に Promise.all で括ってもダメだ。フィルタされない値が返ってきてしまう。

```js
Promise.all(array.filter(async i => (await echo(i)) % 20 === 0)).then(
  console.log
)
// > [ 0, 10, 20, 30, 40, 50, 60 ]
// > 1000ms待機
```

## map との合わせ技

StackOverflow に良い方法があった。前述の Promise.all を使った map を組み合わせてフィルタする方法だ。

<a href="https://stackoverflow.com/questions/33355528/filtering-an-array-with-a-function-that-returns-a-promise" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

```js
Promise.all(array.map(async i => (await echo(i)) % 20 === 0))
  .then(bits => array.filter(i => bits.shift()))
  .then(console.log)
// > 1000ms待機
// > [ 0, 20, 40, 60 ]

// 0 が含まれているので20の倍数でもなんでもなかった...本筋じゃないからまあいいや
```

この処理だと非同期部分を map に負わせることができるので正しい結果が得られる。この方法はよく考えられていてなるほどと思った。分解して処理を追ってみる。

まず非同期の map を行っているが、map の callback に filter の条件を入れている。そうすると配列の**どのインデックスが条件に合致するか**という結果が boolean の配列で返される。

```js
Promise.all(array.map(async i => (await echo(i)) % 20 === 0))
// > [ true, false, true, false, true, false, true ]
```

次にフィルタを掛けるわけだが、注目すべきは `i => bits.shift()` だ。filter の callback では array の要素を i で参照できるが、それは全く使わずに bits の結果だけで判定している。bits は**どのインデックスが条件に合致するか**という boolean 配列なので、それを `shift()` で順番に取得し、判定しているのだ。

```js
.then(bits => array.filter(i => bits.shift()))
```

### async/await で書き直してみる

……一緒だな、これ

```js
;(async () => {
  const bits = await Promise.all(
    array.map(async i => (await echo(i)) % 20 === 0)
  )
  const result = array.filter(i => bits.shift())
  console.log(result)
})()
```

# reduce

## 要素の合計

最後に reduce、全要素の合計を算出してみる。

例によってそのまま実行

```js
const sum = array.reduce((acc, cur) => {
  return acc + echo(cur)
}, 0)
console.log(sum)

// > 0[object Promise][object Promise][object Promise][object Promise][object Promise][object Promise][object Promise]
// > 1000ms待機
```

凄まじい結果が返ってきてしまった。もちろん NG だ。

### async/await で回す

reduce の callback を async/await で装飾し、accumulator の初期値を Promise.resolve(0)とすることで、accumulator（累積値）を全て Promise で回すことができる。

```js
array
  .reduce(
    async (promise, cur) => (await promise) + (await echo(cur)),
    Promise.resolve(0)
  )
  .then(console.log)
// > 7000ms待機
// > 210
```

これで正しい値が取得できた。

## 要素を再配列化

reduce を使って map と同じことしてみる。

```js
array
  .reduce(async (promise, cur) => {
    const acc = await promise
    acc.push((await echo(cur)) * 2)
    return Promise.resolve(acc)
  }, Promise.resolve([]))
  .then(console.log)
// > 7000ms待機
// > [ 0, 20, 40, 60, 80, 100, 120 ]
```

原理は一緒、ただ accumulator のメソッドを呼び出したい場合は一度変数（`acc`）を経由して Promise を resolve する。また、return する時に Promise で再パックする必要がある。ちょいめんどい。

### 7/19 追記

もっと簡単に書けた

```js
array
  .reduce(
    async (promise, cur) => [...(await promise), (await echo(cur)) * 2],
    Promise.resolve([])
  )
  .then(console.log)
```

## 要素をオブジェクト化

同様にオブジェクト化してみる。キーをインデックス、値はそのまま値として、以下のようなものに変換したい。

```json
{
  "0": 0,
  "1": 10,
  "2": 20,
  "3": 30,
  "4": 40,
  "5": 50,
  "6": 60
}
```

配列にする場合とほとんど同じ、違いは引数で index を参照するくらいか

```js
array
  .reduce(async (promise, cur, index) => {
    const acc = await promise
    acc[index] = await echo(cur)
    return Promise.resolve(acc)
  }, Promise.resolve({}))
  .then(console.log)
// > 7000ms待機
// > { '0': 0, '1': 10, '2': 20, '3': 30, '4': 40, '5': 50, '6': 60 }
```

問題なく取得できた。

### 7/19 追記 2

こっちももっと簡単に書けた

```js
array
  .reduce(
    async (promise, cur, index) => ({
      ...(await promise),
      [index]: await echo(cur),
    }),
    Promise.resolve({})
  )
  .then(console.log)
```

# まとめ

| メソッド | やり方                                      |
| -------- | ------------------------------------------- |
| forEach  | `for...of`                                  |
| map      | `Promise.all` + `async/await`               |
| filter   | map(`Promise.all` + `async/await`) + filter |
| reduce   | `async/await` + `Promise.resolve`           |

色々書いているうちに Promise に対する理解が少し深まった気がした。

# 余談

全部書いてから下記を見つけた。これでいいじゃん…

<a href="https://qiita.com/toniov/items/127267fb64a960e8166e" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

<a href="https://github.com/toniov/p-iteration" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

まあ、探せばあるよねぇ。。。

# 参考

- [async/await を、Array.prototype.forEach で使う際の注意点、という話 - Qiita](https://qiita.com/frameair/items/e7645066075666a13063)
- [Array 操作関数の callback に async/await な関数を指定したい - Qiita](https://qiita.com/janus_wel/items/1dc491d866f49af76e98)
- [javascript - Filtering an array with a function that returns a promise - Stack Overflow](https://stackoverflow.com/questions/33355528/filtering-an-array-with-a-function-that-returns-a-promise)
- [promise - JavaScript array .reduce with async/await - Stack Overflow](https://stackoverflow.com/questions/41243468/javascript-array-reduce-with-async-await)
