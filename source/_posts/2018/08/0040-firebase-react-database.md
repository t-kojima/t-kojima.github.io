---
title: '[Firebase x React] ReactをFirebaseでDatabase'
date: 2018-08-13 00:00:56
tags:
  - firebase
  - react
---

Firebase で静的サイトのホスティングと ESLint・Prettier の適用までやったので、Firebase Database を使って定番の ToDo アプリを作ってみたい。

ただ実際に写したり動かしたり色々検討しながらなので、内容がかなり乱暴になっているのはご容赦いただきたい。

---

- 9/24 大幅に書き直し、ちゃんとしました。

---

<!-- more -->

# 前提作業

事前にアプリの雛形の作成と、ESLint・Prettier の設定を行っておく。

- [[Firebase x React] React を Firebase で Hosting](https://t-kojima.github.io/2018/08/12/0038-firebase-react-hosting/)
- [[Firebase x React] ESLint と Prettier を React に適用](https://t-kojima.github.io/2018/08/12/0039-firebase-react-eslint-prettier/)

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
- Todo.jsx : Todo 1 個に対するコンポーネント
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
  render() {
    return (
      <div className="app">
        <section className="hero container is-info">
          <div className="hero-body">
            <h1 className="title">ToDo Application.</h1>
          </div>
        </section>
        <section className="container">
          <TodoList />
        </section>
      </div>
    );
  }
}
```

ちなみに`hero`や`container`などのクラスは[Bulma](https://bulma.io/)を利用している。以下のように CDN で利用できるので非常にカンタンだ。

```html
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/bulma/0.7.1/css/bulma.min.css">
```

最初は[Material-ui](https://material-ui.com/)などのコンポーネントセットの利用も検討したが、パラメータなどが多く、React 初心者にはちょっと辛そうだったので今回は CSS フレームワークにした。

### TodoList.jsx

次に Todo アイテムをリスト保持するコンポーネントを作成する。

まだ Firebase database と繋いでいないのでとりあえず初期値をベタ書きしているが、このコンポーネントは Todo アイテムのリストを状態（state）として保持する。

```jsx
import React, { Component } from 'react';
import Todo from './Todo';

export default class TodoList extends Component {
  constructor() {
    super();
    this.state = {
      todos: [
        {
          id: 1,
          title: 'Todo その1',
          description: 'Todo その1',
          checked: false,
        },
        {
          id: 2,
          title: 'Todo その2',
          description: 'Todo その2',
          checked: false,
        },
      ],
    };
  }
  render() {
    return (
      <ul>
        {this.state.todos.map(todo => (
          <Todo key={todo.id} {...todo} />
        ))}
      </ul>
    );
  }
}
```

Todo コンポーネントを map で作成し、ul 配下に展開している。ちなみに`{...todo}`は要素を全て Todo コンポーネントの props に引き継ぐ記述だ。

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

## Database アクセス用のモジュール作成

Firebase Database へのアクセスを簡易にするためモジュールを作成します。

- firebase/config.js
- firebase/index.js

まずは config.js に Firebase へアクセスする為の設定情報を持たせます。

Firebase Concole にアクセスし、以下のボタンから接続情報を確認する。

![ウェブアプリにFirebaseを追加](/images/38-03.png)

確認できた接続情報を以下のように`firebase/config.js`に持たせる。

```js
export const firebaseConfig = {
  apiKey: '**********',
  authDomain: '**********.firebaseapp.com',
  databaseURL: 'https://**********.firebaseio.com',
  projectId: '**********',
  storageBucket: '**********.appspot.com',
  messagingSenderId: '**********',
};
```

続いて`firebase/index.js`で config を読み出し、初期化したうえで export する。

今回は database のみなので、`firebaseDb`のみを export する。

```js
import firebase from 'firebase';
import { firebaseConfig } from './config';

firebase.initializeApp(firebaseConfig);
export const firebaseDb = firebase.database();
```

これで各コンポーネントからは以下のように呼び出すことで、DB を利用することができるようになる。

```js
import { firebaseDb } from './firebase';
```

## 初期データのロード

初期データのロードということで、Firebase Database に登録されているデータを初回読み込み時にロードする。現時点ではデータを登録していないけど、データ登録処理を入れた時にスムーズにいくよう先にやっとく。

TodoList.jsx を以下のように書き換える。

```jsx
import React, { Component } from 'react';
import Todo from './Todo';
import { firebaseDb } from './firebase';

export default class TodoList extends Component {
  constructor() {
    super();
    this.state = {
      todos: [],
    };
  }
  componentDidMount() {
    firebaseDb
      .ref('todos')
      .on('value', snapshot => this.setState({ todos: snapshot.val() || [] }));
  }
  render() {
    return (
      <ul>
        {Object.entries(this.state.todos).map(([key, value]) => (
          <Todo key={key} id={key} {...value} />
        ))}
      </ul>
    );
  }
}
```

state の todos をただの空配列にし、`componentDidMount`メソッドを追加した。このメソッドで Firebase Database のデータをロードし、state の todos に設定する。

constructor と componentDidMount で処理を分けているのはデータのロードが非同期だからで、コンポーネントマウント時に改めて setState している。

DB のデータを取得するメソッドとして、ここでは`on`を利用している。ざっくり言うとデータに変化がある度にロードする`on`と、一回限りロードする`once`がある。主な違いは以下の通りだ。

### on

on メソッドでは`'value'`とコールバック関数が引数として渡される。

```js
database.ref('***').on('value', callback);
```

ここで渡された callback はリスナーに登録され、データの状態を監視するようになり、データに変化があり次第 callback をキックするようになる。

今回の用途では`snapshot => this.setState({ todos: snapshot.val() || [] })`というコールバックを登録しているので、データに変化がある度にこの関数が実行される。

ちなみに on メソッドは戻り値としてリスナーに登録された callback 自体を返す。リスナーの登録を解除したい場合は、この返された callback を利用して off メソッドを呼んでやればいい。

```js
// onでリスナー登録
const callback = database
  .ref('***')
  .on('value', snapshot => this.setState({ todos: snapshot.val() || [] }));
// 戻り値のcallbackを利用し、offでリスナー解除
database.ref('***').off('value', callback);
```

### once

once メソッドでも`'value'`とコールバック関数が引数として渡されるが、on と異なりリスナーを登録するわけでは無く一度のみデータを取得する。

```js
database.ref('***').once('value', callback);
```

また、on ではコールバックとしてでしか関数を呼べなかったが、once は Promise を返すこともできる。

```js
// これもOK
database
  .ref('***')
  .once('value')
  .then(callback);
```

### id を key にする

もう一つ変化点として、todos のデータの持ち方を変えている。

データをベタ書きしていた時は、todo 1 件のデータは以下のように id を持っているが

```js
{
  id: 1,
  title: 'Todo その1',
  description: 'Todo その1',
  checked: false,
},
```

次のようなデータの持ち方に変更する。（次項参照）

```js
id: {
  title: 'Todo その1',
  description: 'Todo その1',
  checked: false,
},
```

何故かというと、id は Firebase Database が一意に作成するものを使用するようにする為、データの中に id というフィールドを持つのではなく、データのキーとして id を持つようになるからだ。（フィールドとして id を持ってもいいけど、冗長になるので排除する）

そのため、todos を map で回す際は以下のように書き換える。key と value に分離して、props の id には明示的に key を設定してあげるようにする。

```js
{
  Object.entries(this.state.todos).map(([key, value]) => (
    <Todo key={key} id={key} {...value} />
  ));
}
```

## データ登録フォームの作成

次に新規の ToDo タスクを登録するための入力フォームをコンポーネントで作成する。

InputForm.jsx としよう

```jsx
import React, { Component } from 'react';
import { firebaseDb } from './firebase';

export default class InputForm extends Component {
  constructor(props) {
    super(props);
    this.state = {
      title: '',
      desc: '',
    };
    this.onClick = this.onClick.bind(this);
  }

  onClick() {
    firebaseDb.ref(`todos`).push({
      title: this.state.title,
      description: this.state.desc,
      checked: false,
    });
    this.setState({ title: '', desc: '' });
  }

  render() {
    const { title, desc } = this.state;
    return (
      <div className="container">
        <div className="field">
          <label className="label">Title</label>
          <input
            className="input"
            value={this.state.title}
            onChange={e => this.setState({ title: e.target.value })}
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
          {title &&
            desc && (
              <button className="button is-link" onClick={this.onClick}>
                Submit
              </button>
            )}
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
    firebaseDb.ref(`todos`).push({
      title: this.state.title,
      description: this.state.desc,
      checked: false,
    });
    this.setState({ title: '', desc: '' });
  }
```

`.push()`を使ってデータ登録することにより、Todo に対して一意の ID を付与しつつ登録することができる。

そしてこのコンポーネントを App.js から呼びだす。

```html
        <section className="container">
          <InputForm />
          <TodoList />
        </section>
```

以下のような画面になるはずだ

![フォームを追加した画面](/images/40-03.png)

## Todo を消せるようにする

最後に Todo を削除できるようにする。

完了のチェックボックスを付けて、チェックしたら削除ボタンが押せるようになる。だと Todo リストっぽい（よね？）

以下のように Todo コンポーネントに処理を追加する。

```jsx
import React, { Component } from 'react';
import { firebaseDb } from './firebase';

export default class Todo extends Component {
  constructor() {
    super();
    this.handleCheck = this.handleCheck.bind(this);
    this.handleDelete = this.handleDelete.bind(this);
  }

  handleCheck() {
    firebaseDb.ref(`todos/${this.props.id}`).update({
      checked: !this.props.checked,
    });
  }

  handleDelete() {
    firebaseDb.ref(`todos/${this.props.id}`).remove();
  }

  render() {
    return (
      <li className="todo">
        <nav className="panel">
          <div className="panel-heading">
            <p>{this.props.title}</p>
          </div>
          <div className="panel-block">
            <label className="checkbox">
              <input
                type="checkbox"
                onChange={this.handleCheck}
                checked={this.props.checked}
              />
              {this.props.description}
            </label>
          </div>
          <div className="panel-block">
            {this.props.checked && (
              <button className="button is-link" onClick={this.handleDelete}>
                Delete
              </button>
            )}
          </div>
        </nav>
      </li>
    );
  }
}
```

### チェックボックスを付ける

チェックボックスを追加する。チェック状態は`this.props.checked`と同期させ、クリックすると`handleCheck`という関数を呼ぶようにする。

```html
<input
  type="checkbox"
  onChange={this.handleCheck}
  checked={this.props.checked}
/>
```

呼ばれる関数は以下のものだ

```js
  handleCheck() {
    firebaseDb.ref(`todos/${this.props.id}`).update({
      checked: !this.props.checked,
    });
  }
```

props の id をキーとして todos からデータを指定し、checked のみを update かける。ここで update を掛けると DB の変更を TodoList.jsx で検知するので、checked が変更されたデータが自動的にロードされる。

これでチェックボックスをクリックする度にチェックが切り替わる動作ができるようになる。

### 削除ボタンを付ける

次に、チェックボックスが ON になったら表示される削除ボタンを追加する。

```jsx
{
  this.props.checked && (
    <button className="button is-link" onClick={this.handleDelete}>
      Delete
    </button>
  );
}
```

チェックボックスと同様に、クリックすると`handleDelete`関数を呼ぶ。

```js
  handleDelete() {
    firebaseDb.ref(`todos/${this.props.id}`).remove();
  }
```

props の id をキーとして DB からデータを削除する。これも update と同様に、データの変更を検知して、画面表示の一覧が自動的に更新される。

# さいごに

Todo リストとして必要最小限の機能は実装することはできたが、React も Firebase も**ギリギリ動かせた**というのが正直なところ。

しかしこのお手軽さでリアルタイムな Web アプリを作れるのは非常に可能性を感じる。もっと色々いじり倒してみたいものだ。

# 環境

- Windows 10
- React 16.4.2
- create-react-app 1.5.2
- Firebase 5.3.1

# 参考

- [【React】ToDo アプリを作ってみよう【前編】 - Qiita](https://qiita.com/mikan3rd/items/20152cdd63a708264a9e)
- [React + Redux + Firebase で作る Todo App - Qiita](https://qiita.com/gonta616/items/278a7e81a8b624d9621e#firebase%E3%81%A7%E3%83%97%E3%83%AD%E3%82%B8%E3%82%A7%E3%82%AF%E3%83%88%E3%82%92%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B)
- [r-park/todo-react-redux: Todo app with Create-React-App • React-Redux • Firebase • OAuth](https://github.com/r-park/todo-react-redux)
- [ウェブでのデータの読み取りと書き込み  |  Firebase Realtime Database  |  Firebase](https://firebase.google.com/docs/database/web/read-and-write?hl=ja#listen_for_value_events)
