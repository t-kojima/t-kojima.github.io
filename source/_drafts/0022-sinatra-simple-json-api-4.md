---
title: '[Sinatra] シンプルなJSON API サーバーに手を加える（JSON Schema）'
date: 2018-06-26 16:16:06
tags:
- ruby
- sinatra
---

前の記事（[[Sinatra] シンプルな JSON API サーバーに手を加える（RSpec でテスト）](https://t-kojima.github.io/2018/06/26/0021-sinatra-simple-json-api-3/)）に続いて、JSON Schema でデータのバリデーションをしてみる

#### おしながき

- POST でデータを登録できるようにする
- RSpec でテストする
- **データを JSON Schema でバリデーションする**

<!-- more -->

## JSON Schema とは

JSON Schema とは、ある JSON が正しい構造・仕様を満たしているかを検証する仕組みで、JSON Schema 自体が JSON で記述される。

とりあえず最低限のスキーマとそれを満たす JSON を見てみる

### 例 1

```json
{
  "type": "object",
  "properties": {
    "hoge": {
      "type": "integer"
    }
  }
}
```

```json
{
  "hoge": 3
}
```

まず一番外側にオブジェクト（`"type": "object"`）があり、数値（`"type": "integer"`）の値を持つ `"hoge"` をプロパティとして持つ、と読める。

なので以下のような JSON は数値が入るべきところを文字が入っているのでバリデーションで弾かれる。

```json
{
  "hoge": "3"
}
```

### 例 2

一番外側はオブジェクトじゃなくてもいい、配列（`"type": "array"`）の場合はこうだ

```json
{
  "type": "array",
  "items": {
    "type": "integer"
  }
}
```

```json
[1, 2, 3]
```

たったこれだけの内容でも、愚直にコードでバリデーションを掛けるとわりと大変だ（ハッシュを再帰的に走査してプロパティを一個ずつチェックするとか…？）そこでターゲットの JSON に対し定義済みのスキーマを被せてバリデーションするイメージの JSON Schema は簡単で分かりやすい。

上記は本当に最低限動くサンプルでしかないので、もう少し実用的な記法は以下を参照

- [JSON Schema と友達になる為の記事 - Qiita](https://qiita.com/arumi8go/items/a9530cbd39ff545a7bbb)
- [json schema のよく利用する機能まとめ - Qiita](https://qiita.com/dorachan1029/items/57b86116ae67e94ee1ff)

`JSON Schema の書き方については一応本稿の趣旨ではないので…外部に丸投げする`

また、バリデーションのチェックは [JSON Schema Validator](https://www.jsonschemavalidator.net/) のような Web のツールを使うと楽だ

## Ruby でやるには

[ruby-json-schema/json-schema](https://github.com/ruby-json-schema/json-schema) という gem を使う。この gem では [JSON Schema Draft 4](https://tools.ietf.org/html/draft-zyp-json-schema-04) を使用しているので、以後 Draft 4 に準ずる形で進める。

ちなみに登録するデータは

```json
{
  "title": "すごいHaskellたのしく学ぼう!",
  "author": "Miran Lipovaca",
  "pages": 391
}
```

で、利用するスキーマは

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "description":
    "This is book registration schema, it validates when a post on /books.",
  "required": ["title", "author", "pages"],
  "type": "object",
  "properties": {
    "title": {
      "type": "string"
    },
    "author": {
      "type": "string"
    },
    "pages": {
      "type": "integer"
    }
  },
  "additionalProperties": false
}
```

になる。

### 単純に弾くだけ

単にエラーを弾くだけなら post メソッドへは一行追加するだけでいい（スキーマは書かなきゃだけど…）

```ruby
SCHEMA = {
  "$schema" => "http://json-schema.org/draft-04/schema#",
  description: "This is book registration schema, it validates when a post on /books.",
  required: ["title", "author", "pages"],
  type: "object",
  properties: {
    title: { "type": "string" },
    author: { "type": "string" },
    pages: { "type": "integer" }
  },
  "additionalProperties": false
}

post '/books', provides: :json do
  params = JSON.parse request.body.read

  # この行を追加
  halt 500 unless JSON::Validator.validate(SCHEMA, params)

  book = Book.new do |b|
    b.title = params[:title]
    b.author = params[:author]
    b.pages = params[:pages]
  end
  book.save
end
```

これでバリデーションに失敗した場合 500 エラーになる。

### メッセージも通知する

単純に弾くだけだと何で弾かれたかわからないので、500 にメッセージを乗せてあげる

`JSON::Validator.validate` ではなく `JSON::Validator.fully_validate` を使う。fully_validate を使うと、true/false ではなく 空配列/エラーメッセージ入り配列 で結果を返すようになる。

```ruby
SCHEMA = {
  # 省略
}

post '/books', provides: :json do
  params = JSON.parse request.body.read

  # この2行を追加
  messages = JSON::Validator.fully_validate(SCHEMA, params)
  halt 500, messages.to_json unless messages.empty?

  book = Book.new do |b|
    b.title = params[:title]
    b.author = params[:author]
    b.pages = params[:pages]
  end
  book.save
end
```

これで具体的にどこが弾かれたか通知されるようになった。

## さいごに

Sinatra において、POST の Body（JSON）のバリデーションを JSON Schema で行うことができた。

今回は Sinatra にベタ書いているけど、例えば GET で返す JSON をモデルから自動生成したり
（[r7kamura/json_world](https://github.com/r7kamura/json_world)）、Rack レベルでバリデーションを掛けたり（[r7kamura/rack-json_schema](https://github.com/r7kamura/rack-json_schema#rackjsonschemarequestvalidation)）もできるらしい。

そもそもボクが JSON Schema を知ったきっかけが[全てが JSON になる - ✘╹◡╹✘](http://r7kamura.hatenablog.com/entry/2014/06/10/023433)というブログで、上記のライブラリもこの作者さんのものなんだが、このライブラリ群を見てるとマジで「全てが JSON になる」と思えるようになる。マジですごい（小並感）

## 実行環境

- Ruby 2.3.3
- Sinatra 2.0.3
- json-schema 2.8.0

## 参考

- [JSON スキーマはじめの一歩 - Qiita](https://qiita.com/sagaraya/items/115c15c0df3e84ecbc7f)
- [JSON Schema と友達になる為の記事 - Qiita](https://qiita.com/arumi8go/items/a9530cbd39ff545a7bbb)
- [json schema のよく利用する機能まとめ - Qiita](https://qiita.com/dorachan1029/items/57b86116ae67e94ee1ff)
- [僕が考えた最強の API ドキュメント生成 - 銀の人のメモ帳](https://gin0606.hatenablog.com/entry/2016/02/16/144910)
- [JSON Schema Draft v.4 の規格書を読む](http://blog.wktk.co.jp/ja/entry/2016/01/19/json-schema-draft-v4)
