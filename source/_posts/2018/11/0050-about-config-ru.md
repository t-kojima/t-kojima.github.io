---
title: '[Sinatra] config.ruとは'
date: 2018-11-29 10:09:54
categories:
  - Sinatra
tags:
  - ruby
  - sinatra
---

Sinatra のチュートリアルやブログなど参照していると、手順として「`config.ru`というファイルを作れ」とあるが
何故必要なのか説明できていなかったので、改めて調べてみた。

<!-- more -->

# config.ru を使わないで Sinatra を動かす

例えば Sinatra のみを bundler でインストールする。

```bash
$ mkdir sinatra-test
$ cd sinatra-test
$ bundle init
# Gemfileができるので、gem 'sinatra'を追記
$ bundle install --path vendor/bundle
```

そして以下のようなファイル index.rb を作成し

```ruby
require 'sinatra'
require 'json'

get '/' do
  content_type :json
  response = {
    body: 'welcome to sinatra',
  }
  response.to_json
end
```

以下のコマンドで立ち上げれば、Sinatra 自体は立ち上がる（WEBrick で）

```bash
$ bundle exec ruby index.rb
WARNING: If you plan to load any of ActiveSupport's core extensions to Hash, be
sure to do so *before* loading Sinatra::Application or Sinatra::Base. If not,
you may disregard this warning.
[2018-11-29 10:41:57] INFO  WEBrick 1.4.2
[2018-11-29 10:41:57] INFO  ruby 2.5.1 (2018-03-29) [x86_64-linux]
== Sinatra (v2.0.4) has taken the stage on 4567 for development with backup from WEBrick
[2018-11-29 10:41:57] INFO  WEBrick::HTTPServer#start: pid=23960 port=4567
```

Sinatra 公式でも最小の起動方法はこれであると説明されている。

では config.ru は何に必要なんだろう？

# 自動生成した config.ru を読む

Rails の場合 config.ru が自動生成されるので、それを見てみよう。

まずテスト用のディレクトリを作り、`bundle init`する。

```bash
$ mkdir rails-test
$ cd rails-test
$ bundle init
```

生成された Gemfile の rails のコメントを外し…

`rails new`まで実行する。

```bash
$ bundle install --path vendor/bundle
$ bundle exec rails new .
```

するとデフォルトのファイル群が生成され、以下の内容の config.ru を得ることができる。

```ruby
# This file is used by Rack-based servers to start the application.

require_relative 'config/environment'

run Rails.application
```

このコメントには、「Rack ベースのサーバーでアプリケーションを開始する時に使う」と書いてある。

# Rack とは

Rack について「Rack は Web サーバを立ち上げるためのインターフェース」とよく説明されるが、全然ピンとこない。

- [Rack とは何か - Qiita](https://qiita.com/k0kubun/items/248395f68164b52aec4a)
- [Rack の話 | 初心者向け | DoRuby](https://doruby.jp/users/bibio/entries/Rack__)

上記サイトにちょっとピンと来る説明があった。以下引用

> Rack は、指定したファイルを独自の Ruby DSL として読み込み、DSL で指定した様々なミドルウェア、アプリケーションを組み合わせて Web サーバを立ち上げることができる rackup というコマンドを提供するライブラリである。

この「指定したファイル」というのが config.ru だろう。config.ru を DSL として読み込み、Web サーバを立ち上げる。までが Rack がやってくれることっぽい。

まず config.ru は Rack のファイルだったんだな、そこから分かってなかったわ。

## config.ru は必要？

また、興味深い記述もあった。またまた引用

> require "sinatra"すると at_exit で Sinatra::Application.run!がフックされる。
> Rails::Server と違って Sinatra::Application は Rack::Server のサブクラスではないので、new の際に Rack::Builder DSL を呼んだり、run!の際に Rack::Handler を探したりする実装が含まれている。

つまり一番始めに config.ru 無しで動かした Sinatra サンプルでも、`require "sinatra"`としているので裏では Rack が動き WEBrick が動いていたということだ。

じゃあ config.ru はいらないんじゃないの？

## Rack ミドルウェアを使う場合

Rack ミドルウェアを使う場合、config.ru に設定することで Sinatra でそれらを使うことができる。

Rack ミドルウェアとはどんなものがあるだろう。と思って調べてみたがイマイチ要領を得ない。多分何かの目的で使おうと思っていた gem が Rack ミドルウェアだった！ということはあっても、逆に Rack ミドルウェアを探したら該当の gem がヒットした！ということは無さそうだ。

しかし Sinatra にはロギング・デバッグ・ルーティング・セッション等の標準的なミドルウェアは組み込まれているという。

## いつ使うのか

Sinatra 公式に「config.ru はいつ使うのか？」とドンピシャのセクションがある。

- [Sinatra: README (Japanese)](http://sinatrarb.com/intro-ja.html)

それによると異なる Rack ハンドラ（Unicorn など）を使った時・Sinatra::Base の複数のサブクラスを使う時・Sinatra をミドルウェアとして利用する時、これらの場合は config.ru を使うとしている。

つまり普通に小規模な Web アプリ（クラシックスタイル）で使う場合は、気にしなくていいんじゃないの？

# まとめ

クラシックスタイルで Unicorn/Heroku を使わないなら config.ru は無くても良い。

今の所この認識で問題無さそうだ。しかしクラシックスタイル・モジュラースタイルや Rack ミドルウェアの理解がまだあやふやな部分はかなりある。今後使いながらこの辺も徐々に固めていきたい。

# 参考

- [Sinatra: README (Japanese)](http://sinatrarb.com/intro-ja.html)
- [Rack とは何か - Qiita](https://qiita.com/k0kubun/items/248395f68164b52aec4a)
- [Rack の話 | 初心者向け | DoRuby](https://doruby.jp/users/bibio/entries/Rack__)
