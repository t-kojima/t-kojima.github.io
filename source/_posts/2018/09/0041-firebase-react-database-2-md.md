---
title: '[Firebase x React] ReactをFirebaseでDatabase（続き）'
date: 2018-09-08 10:25:20
tags:
  - firebase
  - react
---

[前回](https://t-kojima.github.io/2018/08/13/0040-firebase-react-database/)の続きです。

ちなみに完成形のリポジトリは[ここにあります](https://github.com/t-kojima/tutorial-firebase-react)

<!-- more -->

# データベースの変更をフックする。

前回の状態では、リロードするまで一覧のデータが更新されていなかった。

これをデータベースに変更があった時、それを検知して自動的に一覧のデータを更新するようにしたい。とはいえこれは簡単に変更することができる。

TodoList.jsx のココを…

```js
  componentDidMount() {
    firebaseDb
      .ref('todos')
      .once('value')
      .then(snapshot => this.setState({ todos: snapshot.val() || [] }));
  }
```

こうするだけだ

```js
  componentDidMount() {
    firebaseDb
      .ref('todos')
      .on('value', snapshot => this.setState({ todos: snapshot.val() || [] }));
  }
```

`once`を`on`に変更して、結果を`then`で受けていたところを、`on`の引数としてコールバック関数自体を渡すようにする。

簡単に言うと、`once`は一度のデータ取得のみを行うが、`on`はデータが変更される度にコールバック関数を呼ぶような動きをする。

こうすることで、DB の`todos`に変化があるたびに、`snapshot => this.setState({ todos: snapshot.val() || [] })`が呼び出され、画面の一覧データが更新される。

今思うと最初っから`on`にしとけば良かった。

# Todo を消せるようにする

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

## チェックボックスを付ける

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

## 削除ボタンを付ける

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


