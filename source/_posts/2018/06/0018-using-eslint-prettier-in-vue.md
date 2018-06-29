---
title: '[VSCode] ESLintとPrettierをVueプロジェクトに適用する'
date: 2018-06-22 13:41:53
tags:
- eslint
- prettier
- vscode
- vue
---

vue-cli でテンプレートからプロジェクトを作成すると、大体 ESLint のプラグインを自動でチョイスしてくれて設定ファイルまで用意してくれている。そこに乗っかっていると非常に楽なんだけど、どこがどう動いているか理解する為に 0 から自力で環境を作ってみる。

<!-- more -->

vue-cli の webpack-simple テンプレートを基に ESLint(+Prettier)環境を作っていく。

## プロジェクト作成

vue の webpack-simple テンプレートからプロジェクトを作成する。

```bash
yarn vue init webpack-simple .
```

テンプレートを読み込むと、main.js と App.vue が作成される。
この 2 つのファイルに eslint と format を当てることをゴールとしよう。

```
// main.js

import Vue from 'vue'
import App from './App.vue'

new Vue({
  el: '#app',
  render: h => h(App)
})
```

```
<!-- App.vue -->

<template>
  <div id="app">
    <img src="./assets/logo.png">
    <h1>{{ msg }}</h1>
    <h2>Essential Links</h2>
    <ul>
      <li><a href="https://vuejs.org" target="_blank">Core Docs</a></li>
      <li><a href="https://forum.vuejs.org" target="_blank">Forum</a></li>
      <li><a href="https://chat.vuejs.org" target="_blank">Community Chat</a></li>
      <li><a href="https://twitter.com/vuejs" target="_blank">Twitter</a></li>
    </ul>
    <h2>Ecosystem</h2>
    <ul>
      <li><a href="http://router.vuejs.org/" target="_blank">vue-router</a></li>
      <li><a href="http://vuex.vuejs.org/" target="_blank">vuex</a></li>
      <li><a href="http://vue-loader.vuejs.org/" target="_blank">vue-loader</a></li>
      <li><a href="https://github.com/vuejs/awesome-vue" target="_blank">awesome-vue</a></li>
    </ul>
  </div>
</template>

<script>
export default {
  name: 'app',
  data () {
    return {
      msg: 'Welcome to Your Vue.js App'
    }
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}

h1, h2 {
  font-weight: normal;
}

ul {
  list-style-type: none;
  padding: 0;
}

li {
  display: inline-block;
  margin: 0 10px;
}

a {
  color: #42b983;
}
</style>
```

## ESLint を当てる

まずデフォルトでも 2 つのファイルはそこそこ整形されてるので、eslint に怒られる形に書き換える。

```
// main.js

import Vue from "vue";
import App from "./App.vue";

new Vue({
  el: "#app",
  render: h => h(App)
});
```

```
// App.vue

<script>
export default {
  name: "app",
  data () {
    return {
      msg: 'Welcome to Your Vue.js App'
    };
  }
};
</script>
```

- シングルクォートをダブルクォートに
- セミコロンを追加

今の段階では何もエラーにならないし、フォーマットを掛けても変わらない。

### ESLint をインストール

eslint-config-standard をインストールする。他にも色々あるけど、これは eslint-config-standard が依存しているパッケージなのでまとめて入れてしまう。

```bash
yarn add --dev eslint eslint-config-standard eslint-plugin-standard eslint-plugin-promise eslint-plugin-import eslint-plugin-node
```

また、VSCode の `ESLint` Extension をインストールして有効にする。

この段階ではまだ何も変化は無い。続いて.eslintrc を作成を作成する。

```js
module.exports = {
  root: true,
  extends: 'standard',
}
```

ここで main.js に対してズラズラっとエラーが出てきた。

![](/images/18-01.png)

主にダブルクォートとセミコロンで怒られている。ちなみに App.vue には一切エラーが出ていない。そもそもコードハイライトも効いていないので、.vue ファイルをちゃんと認識してないみたいだ。

### Vetur をインストール

App.vue にも ESLint を効かせたい。VSCode の `Vetur` Extension をインストールして有効にする。

App.vue にコードハイライトが効くようになったが、ESLint はまだ効いていないみたいだ。

### eslint-plugin-vue をインストール

vue 用の eslint-plugin をインストールする。

```bash
yarn add --dev eslint-plugin-vue
```

有効にするには.eslintrc.js に追記が必要だ

```js
module.exports = {
  root: true,
  extends: ['standard', 'plugin:vue/essential'],
  plugins: ['vue'],
}
```

さらに VSCode で有効にするには設定（settings.json）に以下を追記する。

```json
{
  "eslint.validate": [
    "javascript",
    {
      "language": "vue",
      "autoFix": true
    }
  ]
}
```

これで App.vue にも ESLint が効き、エラーが表示された！

![](/images/18-02.png)

## Prettier を当てる

ESLint を当てることができたので、次はフォーマッターだ。今エラーが出ている個所をフォーマッターが自動で修正してくれたらイケてますよね？

ところが VSCode 上でフォーマットを掛けても main.js は何も変わらないし、App.vue に至ってはシングルクォートがダブルクォートに変換されてしまい、逆にエラーが増えてしまった。

```
// App.vue
// 「ドキュメントのフォーマット」をすると悪化してしまう。

<script>
export default {
  name: "app",
  data() {
    return {
      msg: "Welcome to Your Vue.js App"
    };
  }
};
</script>
```

ESLint の設定を変えて `eslint --fix` でフォーマットと自動修正もできるみたいだけど、Prettier を使ってフォーマットをしてみたい

### Prettier をインストール

```bash
yarn add --dev prettier eslint-config-prettier eslint-plugin-prettier
```

eslint-config-prettier は ESLint のフォーマット関連のルールを無効にし、eslint-plugin-prettier で ESLint 上で Prettier を動くようにする。つまり ESLint のフォーマット機能を Prettier に変更するということだろう。

.eslintrc.js の修正も必要だ

```js
module.exports = {
  root: true,
  extends: ['standard', 'plugin:vue/essential', 'plugin:prettier/recommended'],
  plugins: ['vue'],
  rules: {
    'prettier/prettier': [
      'error',
      {
        semi: false,
        singleQuote: true,
        trailingComma: 'es5',
      },
    ],
  },
}
```

さらに VSCode に `Prettier` Extension をインストールし、 設定で prettier.eslintIntegration を有効にする。

```json
"prettier.eslintIntegration": true,
```

そして main.js にフォーマットを掛けると、エラーが軒並み消えるようになる。（ついでに.eslintrc.js 自体もフォーマットが掛かるようになる）

しかし main.js にはエラーが 1 個残るだろう、これはフォーマッタで自動修正できないエラーだ。

```
Do not use 'new' for side effects. (no-new)
```

これは修正できず不可避なので、コメントで無効化する。

```
import Vue from 'vue'
import App from './App.vue'

/* eslint-disable no-new */
new Vue({
  el: '#app',
  render: h => h(App),
})
```

ちなみに vue の webpack テンプレートを使うと最初からこの状態の main.js が生成される。

これで完了！と思いきや、App.vue にフォーマットを掛けても修正されない。以下のコマンドで`eslint --fix`を掛けるとキレイに直るので VSCode 側の問題のようだ

```bash
yarn eslint src/App.vue --fix
```

### Prettier は Vetur で動かない

動かないらしい

<a href="https://github.com/prettier/prettier-vscode/issues/338" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

VSCode の設定で、vetur.format.defaultFormatter.js が prettier になっているが

```json
"vetur.format.defaultFormatter.js": "prettier",
```

prettier 側の設定で無効化されているようだ

```json
"prettier.disableLanguages": [
  "vue"
],
```

ただし以下の設定をしておくと`<script>`タグ内には効かないが`<style>`タグ内や html 自体のフォーマットは多少効くようだ

```json
"prettier.singleQuote": true,
"vetur.format.defaultFormatter.html": "js-beautify-html",
```

`<script>`タグ部分は`eslint --fix`を自動で行う設定で対応する。

```json
"eslint.autoFixOnSave": true,
```

これは VSCode の自動保存では効かないが、手動で`Ctrl + S`すると効くので逆に丁度いい。

## 感想

すんげえめんどくさい。とにかく登場人物が多すぎる

- eslint
- eslint-config-standard
- eslint-plugin-standard
- eslint-plugin-promise
- eslint-plugin-import
- eslint-plugin-node
- eslint-plugin-vue
- prettier
- eslint-config-prettier
- eslint-plugin-prettier
- eslint extension
- prettier extension
- vuter extension

例えば python なら、lint モジュールと公式の python 拡張だけで済んでしまう。

- pylint (もしくは autopep8 とか flake8 とか…)
- python extension

そもそも babel+webpack の開発環境の時点でかなりしんどいので、その上に成り立ってる vue で環境構築がしんどいのは必然なのかもしれない…

まあ、その為のプロジェクトテンプレートなんだろうね。ある程度理屈がわかったので、次からは気兼ねなくテンプレートの ESLint 設定に乗っかろう。
