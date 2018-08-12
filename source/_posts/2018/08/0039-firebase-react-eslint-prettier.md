---
title: 0039-firebase-react-eslint-prettier
date: 2018-08-12 10:17:55
tags:
  - firebase
  - react
  - eslint
---

[前回](https://t-kojima.github.io/2018/08/12/0038-firebase-react-hosting/)、create-react-app で React アプリを作成したが必要最小限の構成となっていた。

快適にコーディングするため、まずは ESLint と Prettier を適用させてみたい。

<!-- more -->

# 環境

- VSCode 1.25.1
- Node 8.11.3

エディタは VSCode を使用する。

# ESLint

yarn でガシガシいれていく。まず Standard スタイル

```bash
yarn add --dev eslint eslint-config-standard eslint-plugin-standard eslint-plugin-promise eslint-plugin-import eslint-plugin-node
```

続いて React と JSX

```bash
yarn add --dev eslint-plugin-react eslint-plugin-jsx-a11y
```

そして `.eslintrc.js` を作成する。

```js
module.exports = {
  env: {
    browser: true,
    es6: true
  },
  extends: ["standard"],
  plugins: ["react", "jsx-a11y", "import"],
  rules: {}
};
```

もし VSCode で`ESLint`拡張機能を入れていない場合は入れておこう。

すると ESLint が効くようになるので、js ファイルを開くとエラーが表示されるようになる。

# Prettier

このままだとエラーは表示されるが自動的にフォーマットは掛けてくれない。一個一個手動でエラーを潰すのは手間なので、 Prettier を導入してフォーマットすることで解決できるエラーはこれで潰すと楽だ。

```yarn
yarn add --dev prettier eslint-config-prettier eslint-plugin-prettier
```

## .eslintrc.js

同時に`.eslintrc.js`も修正する。

```js
module.exports = {
  env: {
    browser: true,
    es6: true
  },
  extends: ["standard", "prettier/react", "prettier/standard"],
  plugins: ["react", "jsx-a11y", "import", "prettier"],
  globals: {
    it: false
  },
  rules: {
    "react/jsx-uses-vars": 1,
    "react/jsx-uses-react": 1,
    "space-before-function-paren": 0,
    "prettier/prettier": [
      "error",
      {
        semi: false,
        singleQuote: true,
        trailingComma: "es5"
      }
    ]
  }
};
```

追加のルールには prettier のフォーマットを指定するルール他、3 つ追加している。

### react/jsx-uses-vars

このルールが適用されると、以下の JSX で`import App from './App'`が`no-unused-vars`でエラーとなってしまう。

```js
import App from "./App";

ReactDOM.render(<App />, document.getElementById("root"));
```

`'react/jsx-uses-vars': 1`とすることでこのルールを回避する。

### react/jsx-uses-react

JSX 自体を有効にするため、`import React from 'react'`としてインポートする必要があるが、このルールが適用されるとやはり`no-unused-vars`となる。

```js
import React from "react";
```

これも同様に`'react/jsx-uses-react': 1`でルールを回避できる。

### space-before-function-paren

Standard ルールでは`function ()`のように関数名と()の間にスペースが必要になる。

```
  render () { // OK
  render() {  // NG
```

しかし Prettier のフォーマットではここにスペースを開けてはくれない。VSCode の設定を頑張ればスペースを入れれないこともないが、できるだけシンプルに解決したいし、そもそもスペースがあろうがなかろうがあまり気にしていないので、ルール自体を無効化する。

## .prettierrc.js

prettier の挙動をある程度設定するため、`.prettierrc.js`も作成する。

```js
module.exports = {
  semi: false,
  singleQuote: true,
  trailingComma: "es5"
};
```

作成すると言ったが、VSCode の場合は`"prettier.eslintIntegration": true,`という設定でも良い。この設定を true にすると`.eslintrc.js`を参照して Prettier が挙動を決定する。つまり上記の`.eslintrc.js`のように Prettier の挙動をルールとして定義しておけば、`.prettierrc.js`不要で同様の挙動を取るようにすることができる。

こっちのほうがカンタンだ。

# ついでに

`create-react-app`で生成されるコードは以下のように Component を継承しているが、Stateless Functional Component (SFC) に書き直してみる。

```js
class App extends Component {
  render() {
    return (
      <div className="App">
        <header className="App-header">
          <img src={logo} className="App-logo" alt="logo" />
          <h1 className="App-title">Welcome to React</h1>
        </header>
        <p className="App-intro">
          To get started, edit <code>src/App.js</code> and save to reload.
        </p>
      </div>
    );
  }
}

export default App;
```

とはいえ以下のように関数とするだけだ

```js
export default () => (
  <div className="App">
    <header className="App-header">
      <img src={logo} className="App-logo" alt="logo" />
      <h1 className="App-title">Welcome to React</h1>
    </header>
    <p className="App-intro">
      To get started, edit <code>src/App.js</code> and save to reload.
    </p>
  </div>
);
```

それほど変わっていないが、`render`関数が不要になったり、`Component`を import しなくても良かったりするので、よりシンプルになったと思う。

# さいごに

ESLint と Prettier を適用させたことで、スタイルチェックとフォーマットを使うことができるようになった。

これで安心（？）して React が書けるようになったネ！
