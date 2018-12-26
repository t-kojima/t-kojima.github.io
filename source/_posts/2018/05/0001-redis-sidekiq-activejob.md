---
title: '[Rails] RedisとSidekiqとActiveJobで苦しむ'
date: 2018-05-10 00:38:03
categories:
  - Rails
tags:
  - ruby
  - rails
  - redis
  - sidekiq
---

## Rails でバッチ処理をしたい

具体的には動画のエンコードを Rails 上でやりたい。

もちろんリクエストを投げてエンコード結果のレスポンスを返すなんてことは無理だ、だからバッチでやる。Rails でバッチ処理のやり方を調べると、Rails4.2 からは ActiveJob という機能を使ってやるらしい。

<!-- more -->

<a href="https://railsguides.jp/active_job_basics.html" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

冒頭に ActiveJob の目的なるものが書いてある。

&gt;Active Job の主要な目的は、Rails アプリを即席で作成した直後でも使用できる、自前のジョブ管理インフラを持つことです。これにより、Delayed Job と Resque などのように、さまざまなジョブ実行機能の API の違いを気にせずにジョブフレームワーク機能やその他の gem を搭載することができるようになります。バックエンドでのキューイング作業では、操作方法以外のことを気にせずに済みます。さらに、ジョブ管理フレームワークを切り替える際にジョブを書き直さずに済みます。

なるほど、ジョブ管理フレームワーク（アダプタ）の差異を吸収してくれる機能なわけね。

アダプタは情報が多そうな`Sidekiq`を使ってみることにする。

## Redis で苦しむ

`Sidekiq`を使うには`Redis`のインストールが必要なようだ。WSL（Windows Subsystem for Linux）の Ubuntu で動作させる為、Redis を apt からインストールする。

```
sudo apt-get install -y redis-server
```

うまくインストールされた。`redis-server`コマンドで Redis 自体も立ち上がった。次は Sidekiq だ

Gemfile に追記し

```ruby:Gemfile
gem 'sidekiq'
```

インストールする

```
bundle install
```

そして Sidekiq を起動......起動しない

```bash
$ bundle exec sidekiq
2018-03-31T02:45:54.416Z 569 TID-oxysghnw1 INFO: Booting Sidekiq 5.1.1 with redis options {:url=>"redis://127.0.0.1:6379", :namespace=>"sidekiq", :id=>"Sidekiq-server-PID-569"}


         m,
         `$b
    .ss,  $$:         .,d$
    `$$P,d$P'    .,md$P"'
     ,$$$$$bmmd$$$P^'
   .d$$$$$$$$$$P'
   $$^' `"^$$$'       ____  _     _      _    _
   $:     ,$$:       / ___|(_) __| | ___| | _(_) __ _
   `b     :$$        \___ \| |/ _` |/ _ \ |/ / |/ _` |
          $$:         ___) | | (_| |  __/   <| | (_| |
          $$         |____/|_|\__,_|\___|_|\_\_|\__, |
        .d$$                                       |_|

2018-03-31T02:45:59.593Z 569 TID-oxysghnw1 INFO: Running in ruby 2.3.3p222 (2016-11-21 revision 56859) [x86_64-linux]
2018-03-31T02:45:59.593Z 569 TID-oxysghnw1 INFO: See LICENSE and the LGPL-3.0 for licensing details.
2018-03-31T02:45:59.594Z 569 TID-oxysghnw1 INFO: Upgrade to Sidekiq Pro for more features and support: http://sidekiq.org
Error connecting to Redis on 127.0.0.1:6379 (Errno::ECONNREFUSED)
```

`redis-cli`でもダメだ

```
Could not connect to Redis at 127.0.0.1:6379: Connection refused
```

結局原因は分からなかったが、パッケージマネージャからインストールすると Redis のバージョンが古い模様（この時 2.8 がインストールされていた）

### 最新バージョンをインストールする

[Ubuntu Linux 14.04 LTS に Redis 3 をインストールする](http://d.hatena.ne.jp/Kazuhira/20160116/1452933029)を参考に最新バージョンを入れる。

```
sudo add-apt-repository ppa:chris-lea/redis-server
sudo apt-get update
sudo apt-get install redis-server
```

見事 4.0.9 がインストールされた...が、今度は起動しないのだ。

```bash
$ redis-server
68:C 31 Mar 13:27:35.509 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
68:C 31 Mar 13:27:35.509 # Redis version=4.0.9, bits=64, commit=00000000, modified=0, pid=68, just started
68:C 31 Mar 13:27:35.509 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
68:M 31 Mar 13:27:35.509 # You requested maxclients of 10000 requiring at least 10032 max file descriptors.
68:M 31 Mar 13:27:35.509 # Server can't set maximum open files to 10032 because of OS error: Invalid argument.
68:M 31 Mar 13:27:35.509 # Current maximum open files is 2048. maxclients has been reduced to 2016 to compensate for low ulimit. If you need higher maxclients increase 'ulimit -n'.
68:M 31 Mar 13:27:35.509 # Creating Server TCP listening socket *:6379: bind: Address already in use
```

config ファイルの指定をしたら立ち上がった
`--help`には未指定ならデフォルトって書いてるのに...

```
sudo redis-server /etc/redis/redis.conf
```

そして今度こそ Sidekiq が立ち上がった。

```bash
$ bundle exec sidekiq
2018-03-31T04:35:18.081Z 92 TID-oxyufulvw INFO: Booting Sidekiq 5.1.1 with redis options {:url=>"redis://localhost:6379", :namespace=>"sidekiq", :id=>"Sidekiq-server-PID-92"}


         m,
         `$b
    .ss,  $$:         .,d$
    `$$P,d$P'    .,md$P"'
     ,$$$$$bmmd$$$P^'
   .d$$$$$$$$$$P'
   $$^' `"^$$$'       ____  _     _      _    _
   $:     ,$$:       / ___|(_) __| | ___| | _(_) __ _
   `b     :$$        \___ \| |/ _` |/ _ \ |/ / |/ _` |
          $$:         ___) | | (_| |  __/   <| | (_| |
          $$         |____/|_|\__,_|\___|_|\_\_|\__, |
        .d$$                                       |_|

2018-03-31T04:35:22.032Z 92 TID-oxyufulvw INFO: Running in ruby 2.3.3p222 (2016-11-21 revision 56859) [x86_64-linux]
2018-03-31T04:35:22.032Z 92 TID-oxyufulvw INFO: See LICENSE and the LGPL-3.0 for licensing details.
2018-03-31T04:35:22.032Z 92 TID-oxyufulvw INFO: Upgrade to Sidekiq Pro for more features and support: http://sidekiq.org
2018-03-31T04:35:22.053Z 92 TID-oxyufulvw INFO: Starting processing, hit Ctrl-C to stop
```

### Redis を Docker で入れる

ここまでやって「あれ...Docker でいいんじゃね？」と気付く

```
docker run --name redis -d -p 6379:6379 redis redis-server
```

らくちーん！

## Sidekiq で苦しむ

ActiveJob の動作テストをする為に、まず Job を作成する。

```
$ bundle exec rails g job encoder
invoke rspec
identical spec/jobs/encoder_job_spec.rb
create app/jobs/encoder_job.rb
```

Job にはログ出力のみ追記しておく。

```
class EncoderJob < ApplicationJob
  queue_as :default
  def perform(*args)
    Rails.logger.debug('do job!')
  end
end
```

そして`rails runner`で ActiveJob をキックする。

```
bundle exec rails runner 'EncoderJob.perform_later'
```

...ログが出ない

```
[ActiveJob] Enqueued Encoder::Job (Job ID: 2f2f8b9e-ce21-474b-b8d7-152aac0750f2) to Sidekiq(encoder)
[ActiveJob] [Encoder::Job] [2f2f8b9e-ce21-474b-b8d7-152aac0750f2] Performing Encoder::Job (Job ID: 2f2f8b9e-ce21-474b-b8d7-152aac0750f2) from Sidekiq(encoder)
```

Enqueue と Perform のログは出ているので実行はされているようだが、肝心の`Rails.logger.debug('do job!')`が出力されない。
解決... [Rails でバッチ処理を作成してみる（runner の場合）](http://d.hatena.ne.jp/yk5656/20140804/1407572756) `rails runner`でキックする場合`Logger`を`new`しないとダメなんかい

### queue でさらに詰まる

`queue_as :default`を`queue_as :encoder`に変えたらまたログが出なくなった。
起動時に使用するキューの設定を`-q`でする必要があった `bundle exec sidekiq -q default encoder`で OK...ではない`-q`直後のキューしか認識しない
`-q default`なら default しか動かないし、`-q encoder`なら encoder しか動かない `sidekiq.yml`で設定したら両方動いた、なんでだろ...

```
:verbose: false
:pidfile: ./tmp/pids/sidekiq.pid
:logfile: ./log/sidekiq.log
:concurrency:  25
:queues:
  - default
  - downloader
  - encoder
```

```
bundle exec sidekiq -C config/sidekiq.yml
```

これで OK

### ダッシュボードはいい

Sidekiq にはダッシュボードがついている、これはいいものだ、以下の設定で有効になる。

```ruby
# config/routes.rb
require 'sidekiq/web'
mount Sidekiq::Web => '/sidekiq'
```

開発環境なら`http://localhost:3000/sidekiq`にアクセスで表示される。

![ダッシュボードの表示例](/images/01-01.png)

### アダプタ意識しなくていい？

冒頭で ActiveJob はジョブ管理フレームワーク（アダプタ）の差異を吸収すると書いたが、結局 Sidekiq がっつり覚えなきゃ最低限の動作も難しかった。
アダプタを変更しても ActiveJob より先は修正が必要ないという事だろうけど、アダプタの変更ってそんな発生するものだろうか？ActiveJob のメリットが感じ辛かった、Sidekiq 単体でいいんじゃないかと。

全くの 0 から導入する場合、`ActiveJob+Sidekiq+Redis`と`Sidekiq+Redis`なら明らかに後者の負荷が低いだろう。
