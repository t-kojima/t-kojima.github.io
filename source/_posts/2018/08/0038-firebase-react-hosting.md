---
title: '[Firebase] ReactをFirebaseでHosting'
date: 2018-08-12 10:00:21
tags:
  - firebase
  - react
---

2 ヶ月前ほどに Firebase で HelloWorld 的なチュートリアルはやったんだけど、2 か月経ったらほとんど忘れた＆触れてない機能がまだまだあるので、ステップバイステップでミニマルに触ってみたいと思う。

全然関係無いけど StepByStep と minimal ってワードが好きです。

<!-- more -->

# 環境

- Node 8.11.3
- Firebase CLI 4.0.3

# 静的サイトをホスティング

まず一番基本的（？）な静的サイトのホスティングをしてみたい。アップするサイトは html 一つでもいいが、ここは react の HelloWorld も一緒にやってしまう。

## React で HelloWorld

[create-react-app](https://github.com/facebook/create-react-app)を使ってサクっと React プロジェクトを作成する。

```bash
mkdir tutorial-firebase-react
cd tutorial-firebase-react

yarn init -y
yarn global add create-react-app
create-react-app .
```

プロジェクトディレクトリを作成し、`create-react-app`を yarn でグローバルインストールし、`.`(カレントディレクトリ)を対象に React アプリを作成する。

いつもは`yarn add --dev`でこういったツールを入れる（なるべくグローバルに入れたくない）のだが、create-react-app はそうするとカレントディレクトリを対象にアプリを作成できない。create-react-app 自身がインストールされた node_modules や package.json を上書きできない（と、言われる）ためだ。

ちなみに`vue-cli`の場合はローカルインストールでも問題ない。`vue init webpack .`などやるとカレントの node_modules や package.json を上書きするからだ。個人的にはこっちの動きのほうがありがたい。しかしまあ好みの差かな。

インストールが完了したら`yarn start`でローカルサーバが立ち上がり、[http://localhost:3000/](http://localhost:3000/)からアプリを確認できる。（HelloWorld とは表示されないけど、まあそういう意味だ）

## Firebase プロジェクトの準備

まず[Firebase](https://firebase.google.com/)のコンソールにログインする。Google アカウントを持っていない場合は事前に作っておこう。

![右上のボタンからログイン](/images/38-01.png)

「プロジェクトを追加」ボタンからプロジェクトの作成に進む。

![プロジェクトを追加を選択](/images/38-02.png)

とりあえずプロジェクト名「tutorial-react」で作成

するとプロジェクトの管理画面に入れる。入れたらまずは OK

## Firebase のインストール

### ログイン

続いて Firebase 関係のツールをインストールする。まず CLI ツールから入れる。

```bash
yarn global add firebase-tools
```

そして Firebase にログインする。

```bash
firebase login
```

この時初回ログインだとブラウザが立ち上がり、google アカウントでのログインを求められる。ログインして「`Woohoo! Firebase CLI Login Successful`」というメッセージが表示されれば OK だ。コンソールにも以下のメッセージが表示され、ログインできたことが分かる。

```bash
✔  Success! Logged in as **********@gmail.com
```

二度目移行のログインだと前回のログイン情報を利用してログインする。アカウントを切り替えたい場合は`firebase login --reauth`とすることで、再度ブラウザからログイン処理を行う流れになる。

### 初期化

続いてアプリに対し Firebase を初期化する。

```bash
firebase init
```

![Hostingを選択](/images/38-04.png)

今回はまず Hosting のみを選択。最初から色々入れると混乱してしまうので…

他にも色々聞かれるので基本 Enter で良いが、`? What do you want to use as your public directory?`のみは`build`を指定しよう。React アプリのビルド先が`build`ディレクトリだからだ。

### ビルド & デプロイ

最後に React を build して deploy する。

```bash
yarn build
```

ビルドして…

```bash
$ firebase deploy

=== Deploying to 'tutorial-react-79436'...

i  deploying hosting
i  hosting: preparing build directory for upload...
✔  hosting: 11 files uploaded successfully

✔  Deploy complete!

Project Console: https://console.firebase.google.com/project/tutorial-react-*****/overview
Hosting URL: https://tutorial-react-*****.firebaseapp.com
```

デプロイする。

最後にホスティングされた URL が表示されるので、アクセスして React の HelloWorld が表示されれば OK だ。

カンタン！

# おわりに

React は create-react-app のおかげで煩雑な環境構築が必要なく、Firebase もまず Hosting だけなら難しいことは何も無かった。

ただ package.json を確認すると`react`、`react-dom`、`react-scripts`のみがインストールされているので、本当に最低限の構成ということが分かる。それなりに動かすアプリを作成する場合は明らかに機能は不足するので、ベースとして使うにしてもそこそこの肉付けは必要だろう。
