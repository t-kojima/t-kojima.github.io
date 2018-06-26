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

[全てが JSON になる - - r7kamura - はてなブログ](http://r7kamura.hatenablog.com/entry/2014/06/10/023433)
