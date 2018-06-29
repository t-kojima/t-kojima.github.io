---
title: '[Sinatra] シンプルなJSON API サーバーに手を加える（POSTでDB登録）'
date: 2018-06-26 08:48:14
tags:
- ruby
- sinatra
- mysql
---

前の記事（[[Sinatra] シンプルな JSON API サーバーを作る](https://t-kojima.github.io/2018/06/18/0015-sinatra-simple-json-api/)）で非常に簡単な API サーバーを作ったが、そこに少し手を加えてより実用的にしてみたい。

#### おしながき

- **POST でデータを登録できるようにする**
- Rspec でテストする
- データを JSON Schema でバリデーションする

<!-- more -->

## POST でデータ登録

まずは前回と同様に GET でデータベースアクセスできるところまで持っていく。

bundler で gem を入れるが、今回は mysql でやるので `activerecord`と`mysql2`を入れる。

```rb
# Gemfile

gem 'sinatra'
gem "activerecord"
gem "mysql2"
```

`db/database.rb` と `db/database.yml` を用意する。

```rb
# database.rb

require 'bundler/setup'
require 'active_record'
require 'mysql2'

ActiveRecord::Base.configurations = YAML.load_file('./db/database.yml')
ActiveRecord::Base.establish_connection(ENV['SINATRA_ENV'].to_sym)

# class
class Book < ActiveRecord::Base
  self.table_name = 'books'
end
```

```yml
default: &default
  adapter: mysql2
  encoding: utf8
  reconnect: false
  database: bookshelf
  pool: 5

development:
  <<: *default
  username: root
  password: mysql
  host: localhost

test:
  <<: *default
  username: root
  password: mysql
  host: localhost

production:
  <<: *default
  username: <******>
  password: <******>
  host: localhost
```

今回は database.yml で接続環境毎に設定を分け、`SINATRA_ENV` 環境変数で切り替えるようにした。開発環境では`export SINATRA_ENV=development`としている。

### ソケットの罠

`index.rb` を作成し、実行したうえで[http://localhost:4567/books/1](http://localhost:4567/books/1)にアクセスする。

```rb
# index.rb

require 'bundler/setup'
require 'sinatra'
require 'json'
require './db/database'

get '/' do
  content_type :json
  response = {
    message: 'welcome to sinatra',
  }
  response.to_json
end

get '/books/:id' do
  content_type :json
  response = {
    record: Book.find(params[:id])
  }
  response.to_json
end
```

しかしアクセスすると以下のエラーになった。

![Mysql2::Error::ConnectionErrorが発生した](/images/20-01.png)

`※ 実際開発していた API サーバーでのキャプチャなので、アドレスが違うのは勘弁してホシイ`

今回開発環境用の DB として docker で mysql サーバーをローカルに立てていた。しかし mysql2 はソケット接続しようとしていてエラーになっているらしい。解決策はここ（[Ruby On Rails で MySQL の Socket エラーが出た時は素直に localhost を 127.0.0.1 に直すのが一番いいと思う](https://qiita.com/benzookapi/items/07b658700a53155a6263)）にある通り、接続先を`localhost`から`127.0.0.1`に変更することで対応した。

```yml
development:
  <<: *default
  username: root
  password: mysql
  host: 127.0.0.1
```

これで正常にアクセスできることは確認できた。

### Hash とシンボルの罠

続いて POST メソッドを追加していく、こんな感じかな

```rb
post '/books', provides: :json do
  params = JSON.parse request.body.read
  book = Book.new do |b|
    b.title = params[:title]
    b.author = params[:author]
    b.pages = params[:pages]
  end
  book.save
end
```

ところがここにデータを POST すると 500 エラーになる

```bash
curl -v -H "Content-type: application/json" -X POST -d '{"title":"すごいHaskellたのしく学ぼう!","author":"Miran Lipovaca","pages":391}' http://localhost:4567/books/

=> 500 Internal Server Error
```

原因は `params[:title]` などが nil になることだった。「Hash のキーはシンボルと文字列を区別する」というすごい単純なオチだったんだけど、ここ（[Hash から文字列でもシンボルでも値を取り出す](https://qiita.com/QUANON/items/169c73425a6bc50dee51)）を参考にシンボルのまま取り出せるようにしてみる。

```rb
params = JSON.parse request.body.read
```

を

```rb
params = JSON.parse(request.body.read).with_indifferent_access
```

にするだけで正常に登録されるようになった。

ちなみに `with_indifferent_access` は本来 `active_support` と `active_support/core_ext` を require しなければ呼べないが、今回は特に意識することなく呼ぶことができている。

事前に `ActiveRecord::Base.configurations` を実行しているのだが、（たぶん）この中で `active_support` を require しているので、後続の処理では明示的に `active_support` を require しなくても `with_indifferent_access` を呼ぶことができる。

## 実行環境

- Ruby 2.3.3
- Sinatra 2.0.3
- activerecord 5.2.0
- mysql2 0.5.1

## 参考

- [【Sinatra】POST メソッドで JSON を受け取りたい - Qiita](https://qiita.com/izumin5210/items/ca f66ece1f67a0fd6a4c)
- [Ruby On Rails で MySQL の Socket エラーが出た時は素直に localhost を 127.0.0.1 に直すのが一番いいと思う - Qiita](https://qiita.com/benzookapi/items/07b658700a53155a6263)
- [Hash から文字列でもシンボルでも値を取り出す - Qiita](https://qiita.com/QUANON/items/169c73425a6bc50dee51)
