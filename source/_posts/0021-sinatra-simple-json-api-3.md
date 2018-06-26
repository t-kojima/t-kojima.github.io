---
title: '[Sinatra] シンプルなJSON API サーバーに手を加える（RSpecでテスト）'
date: 2018-06-26 11:21:02
tags:
- ruby
- sinatra
- rspec
---

前の記事（[[Sinatra] シンプルな JSON API サーバーに手を加える（POST で DB 登録）](https://t-kojima.github.io/2018/06/18/0015-sinatra-simple-json-api-2/)）に続いて、RSpec でテストを実施する。

#### おしながき

- POST でデータを登録できるようにする
- RSpec でテストする
- データを JSON Schema でバリデーションする

<!-- more -->

## RSpec 導入

Gemfile に以下を追記し、`bundle install` する。

```rb
# Gemfile

group :test do
  gem 'rspec'
  gem 'rack-test'
end
```

rspec init で `.rspec` と `spec/spec_helper.rb` を生成する。

```bash
bundle exec rspec --init
```

.rspec に以下を追記、結果が見やすくなる

```
--color
--format documentation
```

### Sinatra 用の設定

生成された `spec/spec_helper.rb` に sinatra 用の設定を追記していく

```rb
require 'rack/test'

RSpec.configure do |config|

  config.expect_with :rspec do |expectations|
    expectations.include_chain_clauses_in_custom_matcher_descriptions = true
  end

  config.mock_with :rspec do |mocks|
    mocks.verify_partial_doubles = true
  end

  config.shared_context_metadata_behavior = :apply_to_host_groups

  # ここから追記
  config.include Rack::Test::Methods
  ENV['SINATRA_ENV'] = 'test'
  ENV['RACK_ENV'] = 'test'
  require './index'
  def app
    Sinatra::Application
  end
end
```

### GET のテスト

まずは GET のみテストしてみる。以下のように `spec/index_spec.rb` ファイルを作成する。

```rb
# index_spec.rb

describe 'get /(root)' do
  before do
    get '/'
  end

  it 'response is ok' do
    expect(last_response).to be_ok
  end

  it "returns 'welcome to sinatra' message" do
    body = JSON.parse(last_response.body)
    expect(body["message"]).to eq 'welcome to sinatra'
  end
end
```

実行すると成功することが確認できる

```bash
> bundle exec rspec

get /(root)
  response is ok
  returns 'welcome to sinatra' message

Finished in 0.11508 seconds (files took 8.2 seconds to load)
2 examples, 0 failures
```

### POST のテスト

POST のテストは DB の登録を含むので、まずテスト用 DB 環境を検める。

- docker でローカルに mysql サーバを立ててテスト用 DB を作る
- [sqlalchemy-migrate](https://github.com/openstack/sqlalchemy-migrate)で DB のマイグレーションをする
- RSpec でテストを行うたびに DB のデータはクリーンアップする

> ここも sqlalchemy-migrate を使うのは、実際の開発で使っているから…  
> なので、DB のリセットに `rake db:reset` は使えない。

#### FactoryBot を使う

テストデータの投入に FactoryBot を使います。

```rb
# Gemfile

group :test do
  gem 'rspec'
  gem 'rack-test'
  gem 'factory_bot' # <= 追加
end
```

`spec/spec_helper.rb` にも以下を追記

```rb
require 'rack/test'
require 'factory_bot'

RSpec.configure do |config|

  # 省略

  config.include FactoryBot::Syntax::Methods
  FactoryBot.find_definitions
end
```

そして定義ファイル（`spec/factories/books.rb`）を作成

```rb
FactoryBot.define do
  factory :book do
    title "すごいHaskellたのしく学ぼう!"
    author "Miran Lipovaca"
    pages 391
  end
end
```

まずは POST を飛ばせるかだけを確認する為、`spec/index_spec.rb` に以下を追記する。

```rb
describe 'post /books' do
  context 'valid params' do
    let(:headers) do
      { 'Content-Type' => 'application/json', 'Accept' => 'application/json' }
    end

    it 'response is ok' do
      post '/books', attributes_for(:book).to_json.to_s, headers
      expect(last_response).to be_ok
    end
  end
end
```

いくつかハマったポイントがある。

- POST を json で飛ばすため、`headers` を与えてやる必要がある。
- FactoryBot の attributes_for はハッシュを返すが、POST の Body は**JSON 形式の文字列**である必要がある。つまり以下のような変換が必要

```rb
# attributes_for(:book)
{ :title => "すごいHaskellたのしく学ぼう!", :author => "Miran Lipovaca", :pages => 391 }

# attributes_for(:book).to_json
{ "title": "すごいHaskellたのしく学ぼう!", "autho": "Miran Lipovaca", "pages": 391 }

# attributes_for(:book).to_json.to_s
'{ "title": "すごいHaskellたのしく学ぼう!", "autho": "Miran Lipovaca", "pages": 391 }'
```

このままレコードが正常登録されるかの spec も追加したいが、このままでは spec 毎に post が走ってレコードがその分登録されてしまう。レコードのカラムにユニーク制約が付いている場合はエラーにもなる。

```rb
    it 'register book' do
      expect do
        post '/books', attributes_for(:book).to_json.to_s, headers
      end.to change(Book, :count).by(1)
    end
```

#### Database Cleaner を使う

上記の問題を解消するため、`database_cleaner` を使う

```rb
# Gemfile

group :test do
  gem 'rspec'
  gem 'rack-test'
  gem 'factory_bot'
  gem 'database_cleaner' # <= 追加
end
```

`spec/spec_helper.rb` に以下追記

```rb
require 'rack/test'
require 'factory_bot'
require 'database_cleaner'

RSpec.configure do |config|

  # 省略

  config.before(:suite) do
    DatabaseCleaner.strategy = :truncation
  end

  config.before(:each) do
    DatabaseCleaner.start
  end

  config.after(:each) do
    DatabaseCleaner.clean
  end
end
```

これで spec 毎に DB のレコードをクリーンアップするようになった。

RSpec の実行結果もこの通りだ

```bash
> bundle exec rspec

get /(root)
  response is ok
  returns 'welcome to sinatra' message

post /books
  valid params
    response is ok
    register book

Finished in 4.79 seconds (files took 9.58 seconds to load)
4 examples, 0 failures
```

## さいごに

今まで Rails でしか RSpec を使っていなかったので、素の RSpec を動かすのがすごいしんどかった。特にテスト用 DB を意識することなく扱えてたのはものすごい Rail に乗ってたんだなというのを実感

## 実行環境

- Ruby 2.3.3
- Sinatra 2.0.3
- RSpec 3.7.0
- FactoryBot 4.10.0
- DatabaseCleaner 1.7.0

## 参考

- [Sinatra のインストールと Rspec でテストする](http://48n.jp/blog/2016/04/20/try-sinatra/)
- [rspec の基本的な使い方 (ruby/sinatra を含む)](https://www.qoosky.io/techs/f20cbb9b26)
- [JSON の POST リクエストと Rspec でテストする方法](http://www.tom08.net/entry/2016/08/03/115143)
- [Sinatra + RSpec + FactoryGirl のスケルトンがやっと出来た](http://j-caw.co.jp/blog/?p=1042)
