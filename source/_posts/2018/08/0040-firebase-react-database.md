---
title: '[Firebase] ReactをFirebaseでDatabase'
date: 2018-08-13 00:00:56
tags:
  - firebase
  - react
---

Firebase で静的サイトのホスティングと ESLint・Prettier の適用までやったので、Firebase Database を使って定番の ToDo アプリを作ってみたい。

ただ実際に写したり動かしたり色々検討しながらなので、内容がかなり乱暴になっているのはご容赦いただきたい。

---

- 9/8 大幅に書き直し、ちゃんとしました。

---

<!-- more -->

# 前提作業

事前にアプリの雛形の作成と、ESLint・Prettier の設定を行っておく。

- [[Firebase] React を Firebase で Hosting](https://t-kojima.github.io/2018/08/12/0038-firebase-react-hosting/)
- [[React] ESLint と Prettier を React に適用](https://t-kojima.github.io/2018/08/12/0039-firebase-react-eslint-prettier/)

まあ ESLint・Prettier の設定は必須では無いけど。。。

## Firebase Database の初期化

まずは Firebase Database を扱う為に、設定ファイルの生成や初期化を行う。

```bash
firebase init database
```

すると`database.rules.json`が作成され、`firebase.json`に database の設定が追記される。

# ToDo アプリの作成

## 構成

ToDo アプリは以下の構成にする。

- index.js : App.js を呼び出すエントリーポイント
- App.js : アプリ全体のレイアウトを持つコンポーネント
- TodoList.jsx : TodoList のコンポーネント
- Todo.jsx : Todo1 個に対するコンポーネント
- InputForm.jsx : Todo を登録するフォームのコンポーネント

### index.js

index.js はエントリーポイントとなるファイルになる。

```js
import React from 'react';
import ReactDOM from 'react-dom';
import './index.css';
import App from './App';
import registerServiceWorker from './registerServiceWorker';

ReactDOM.render(<App />, document.getElementById('root'));
registerServiceWorker();
```

処理としては`#root`の要素に App コンポーネントをレンダリングするのみだ。
ここはある程度お決まりの記述になる。いわゆる*おまじない*だ。

registerServiceWorker は create-react-app が自動生成したモジュールになる。ぶっちゃけ何をしているのか分かっていないので、近いうちに内容をよく確認したい。。。

### App.js

App.js は実質的なルートになるコンポーネントになる。
ここではタイトルを表示し、TodoList コンポーネントを読み込んでいる。

```jsx
import React, { Component } from 'react';
import TodoList from './TodoList';
import './App.css';

export default class App extends Component {
  constructor() {
    super();
    this.state = {
      todos: [
        {
          id: 1,
          title: 'Todo その1',
          description: 'Todo その1',
          done: false,
        },
        {
          id: 2,
          title: 'Todo その2',
          description: 'Todo その2',
          done: false,
        },
      ],
    };
  }
  render() {
    return (
      <div className="app">
        <section className="hero container is-info">
          <div className="hero-body">
            <h1 className="title">ToDo Application.</h1>
          </div>
        </section>
        <section className="container">
          <TodoList todos={this.state.todos} />
        </section>
      </div>
    );
  }
}
```

まだ Firebase database と繋いでいないので、とりあえず初期値をベタ書きしている。

`hero`や`container`などのクラスは[Bulma](https://bulma.io/)を利用している。以下のように CDN で利用できるので非常にカンタンだ。

```html
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/bulma/0.7.1/css/bulma.min.css">
```

最初は[Material-ui](https://material-ui.com/)などのコンポーネントセットの利用も検討したが、パラメータなどが多く、React 初心者にはちょっと辛そうだったので今回は CSS フレームワークにした。

### TodoList.jsx

Todo アイテムをリスト保持するコンポーネントだ。

```jsx
import React, { Component } from 'react';
import Todo from './Todo';

export default class TodoList extends Component {
  render() {
    return (
      <ul>
        {this.props.todos.map(todo => (
          <Todo key={todo.id} {...todo} />
        ))}
      </ul>
    );
  }
}
```

Todo コンポーネントを map で作成し、ul 配下に展開している。ちなみに`{...todo}`は要素を全て Todo コンポーネントに引き継ぐ記述になる。

### Todo.js

最後に Todo 1 個あたりのコンポーネントを作る。

```jsx
import React, { Component } from 'react';

export default class Todo extends Component {
  render() {
    return (
      <li className="todo">
        <nav className="panel">
          <div className="panel-heading">
            <p>{this.props.title}</p>
          </div>
          <div className="panel-block">
            <p>{this.props.description}</p>
          </div>
          <div className="panel-block">
            <a href="">Completed</a>
          </div>
        </nav>
      </li>
    );
  }
}
```

基本的には受け取った props の title や description を表示するだけ。それなりにタグが色々あるように見えるのは Bulma の panel を使っているからだ。

ここで一度実行(`yarn start`)すると以下のような画面が立ち上がる。

![実行画面](/images/40-01.png)

このようにベタ書きではあるが Todo が表示されれば OK だ。

# Firebase Database 適用

ここまでで静的な Todo アプリ（アプリと言ってよいのだろうか。。。？）が出来たが、ここに Firebase database との連携を追加して動きのあるページにしていく。

まず Firebase SDK をインストールする。前に入れた Firebase CLI とは別物で、Firebase Database 等へのアクセスが可能になる。

```bash
yarn add firebase
```

## 初期データのロード

初期データのロードということで、Firebase Database に登録されているデータを初回読み込み時にロードする。現時点ではデータを登録していないけど、データ登録処理を入れた時にスムーズにいくよう先にやっとく。

```jsx
import React, { Component } from 'react';
import TodoList from './TodoList';
import './App.css';
import firebase from 'firebase/app';
import 'firebase/database';

export default class App extends Component {
  constructor() {
    super();
    this.state = {
      todos: [],
    };
  }
  componentDidMount() {
    firebase
      .database()
      .ref('todos')
      .once('value')
      .then(snapshot => {
        this.setState({ todos: Object.values(snapshot.val()) });
      });
  }
  render() {
    return (
      <div className="app">
        <section className="hero container is-info">
          <div className="hero-body">
            <h1 className="title">ToDo Application.</h1>
          </div>
        </section>
        <section className="container">
          <TodoList todos={this.state.todos} />
        </section>
      </div>
    );
  }
}
```

state の todos をただの空配列にし、`componentDidMount`メソッドを追加した。このメソッドで Firebase Database のデータをロードし、state の todos に設定する。

constructor と componentDidMount で処理を分けているのはデータのロードが非同期だからで、コンポーネントマウント時に改めて setState している。

## データ登録フォームの作成

ToDo タスクを登録するための入力フォームをコンポーネントで作成する。

```jsx
import React, { Component } from 'react';
import firebase from 'firebase/app';
import 'firebase/database';

export default class Todo extends Component {
  constructor(props) {
    super(props);
    this.state = {
      text: '',
      desc: '',
    };
    this.onClick = this.onClick.bind(this);
  }

  onClick() {
    const newKey = firebase
      .database()
      .ref('todos')
      .push().key;
    firebase
      .database()
      .ref(`todos/${newKey}`)
      .set({
        id: newKey,
        title: this.state.text,
        description: this.state.desc,
        checked: false,
      });
    this.setState({ text: '', desc: '' });
  }
  render() {
    return (
      <div className="container">
        <div className="field">
          <label className="label">Title</label>
          <input
            className="input"
            value={this.state.text}
            onChange={e => this.setState({ text: e.target.value })}
          />
        </div>
        <div className="field">
          <label className="label">Description</label>
          <input
            className="input"
            value={this.state.desc}
            onChange={e => this.setState({ desc: e.target.value })}
          />
        </div>

        <div className="control">
          <button className="button is-link" onClick={this.onClick}>
            Submit
          </button>
        </div>
      </div>
    );
  }
}
```

入力フォームではタスクの Title と Description を登録できるようにする。

```js
  constructor(props) {
    super(props)
    this.state = {
      text: '',
      desc: ''
    }
    this.onClick = this.onClick.bind(this)
  }
```

コンストラクタでは state の初期化と、イベントの bind を行う。イベントの bind は[React.createClass ではなく React.Component を使う場合に必要](https://qiita.com/cubdesign/items/ee8bff7073ebe1979936)なようだ。

submit の onClick で firebase の API を呼び出す。

```js
 onClick() {
    const newKey = firebase
      .database()
      .ref('todos')
      .push().key
    firebase
      .database()
      .ref(`todos/${newKey}`)
      .set({
        id: newKey,
        title: this.state.text,
        description: this.state.desc,
        checked: false
      })
    this.setState({ text: '', desc: '' })
  }
```

このコンポーネントを App.js から呼びだす。

```html
        <section className="container">
          <InputForm />
          <TodoList todos={this.state.todos} />
        </section>
```

以下のような画面になるはずだ

![フォームを追加した画面](/images/40-03.png)

しかしこのままだとタスクの登録はできても、画面をリロードしなければ一覧に反映されない。次回はデータベースの変更をフックする機構を追加したい。

つづく

# 参考

- [【React】ToDo アプリを作ってみよう【前編】 - Qiita](https://qiita.com/mikan3rd/items/20152cdd63a708264a9e)
- [React + Redux + Firebase で作る Todo App - Qiita](https://qiita.com/gonta616/items/278a7e81a8b624d9621e#firebase%E3%81%A7%E3%83%97%E3%83%AD%E3%82%B8%E3%82%A7%E3%82%AF%E3%83%88%E3%82%92%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B)
- [r-park/todo-react-redux: Todo app with Create-React-App • React-Redux • Firebase • OAuth](https://github.com/r-park/todo-react-redux)
  <!-- - [Vue.js で ToDo アプリを作る (Firebase Realtime Database 編) - kntmr-blog](https://kntmr.hatenablog.com/entry/2018/06/17/093357)
- [playground/vue-pwa-examples/todo-app at master · kntmr/playground](https://github.com/kntmr/playground/tree/master/vue-pwa-examples/todo-app) -->
