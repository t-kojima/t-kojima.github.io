---
title: '[Sinatra] シンプルなJSON API サーバーを作る'
date: 2018-06-18 15:25:16
tags:
- ruby
- sinatra
---

既存システムのデータを他システムでも使いたいという要件は頻繁にあるが、既存システムを改修してデータを飛ばせるようにすることはできない（…わけではないが予算的に無理）

なので自力で API サーバーを立てるお！

<!-- more -->

ちなみに以下サイトを参考にしている。というかほとんど下記サイトのまんま

[sinatra でミニマムな Web サービスを公開してみるよ【sinatra + nginx + puma + Capistrano】](https://tisnote.com/sinatra-web-setup-deploy/)

以下はサンプルで作ったリポジトリ

[https://github.com/t-kojima/sinatra-api-sample](https://github.com/t-kojima/sinatra-api-sample)

## sinatra を使う

限りなくシンプルにやりたいので sinatra を使う。bundler でインストールし、以下のコードだけでもう動いてしまう。

```ruby
# index.rb

require 'bundler/setup'
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

```bash
ruby index.rb
```

これで Webrick が起動するので、[http://localhost:4567](http://localhost:4567)にアクセスすると上記の json を取得できる。簡単すぎる！

## database にアクセス

業種柄（？）既存システムの DB は SQLServer が多いので、`activerecord-sqlserver-adapter`と`tiny_tds`を使って DB にアクセスする。

[https://github.com/t-kojima/ruby-sqlserver-script](https://github.com/t-kojima/ruby-sqlserver-script)

```ruby
require 'bundler/setup'
require 'activerecord-sqlserver-adapter'

ActiveRecord::Base.configurations = YAML.load_file('./db/database.yml')
ActiveRecord::Base.establish_connection(:development)

# class
class Book < ActiveRecord::Base
  self.table_name = '<books>'
end
```

ActiveRecord を継承するので、利用側はいつも通り（？）に使えばいい

こんな感じ

```ruby
get '/books/:id' do
  content_type :json
  response = {
    body: Book.where(id: params[:id])
  }
  response.to_json
end
```

SQLServer 以外の RDB を使う場合はアダプタを変えればいいだけだ

## capistrano でデプロイ

別にサーバー上で手動デプロイしてもいいんだけど、ruby は capistrano で統一しておくと考えるのが楽なので capistrano でやる

以下 gem をインストール

```ruby
gem 'puma'

group :development do
  gem 'capistrano',         require: false
  gem 'capistrano-bundler', require: false
  gem 'capistrano3-puma',   require: false
  gem 'capistrano-rbenv',   require: false
end
```

```bash
bundle exec cap install
```

`Capfile` ができるので以下を追記

```ruby
require 'capistrano/setup'
require 'capistrano/deploy'
require 'capistrano/rbenv'
require 'capistrano/bundler'
require 'capistrano/puma'

install_plugin Capistrano::Puma
```

config/deploy.rb を編集、`#deploy`セクションと`#puma`セクションはよほどじゃなければいじらない

```ruby
# deploy.rb

# config valid for current version and patch releases of Capistrano
lock "~> 3.11.0"

set :application, '<application name>'

# git
set :repo_url, 'https://<repository>.git'
set :deploy_via, :remote_cache

# rbenv
set :rbenv_type, :user
set :rbenv_ruby, '2.3.3'

# deploy
set :deploy_to, "/var/www/#{fetch(:application)}"
set :linked_dirs, fetch(:linked_dirs, []).push(
  'log',
  'tmp/pids',
  'tmp/cache',
  'tmp/sockets',
  'vendor/bundle',
  'public/system',
  'public/uploads'
)

namespace :deploy do
  desc 'Make sure local git is in sync with remote.'
  task :check_revision do
    on roles(:app) do
      unless `git rev-parse HEAD` == `git rev-parse origin/master`
        puts 'WARNING: HEAD is not the same as origin/master'
        puts 'Run `git push` to sync changes.'
        exit
      end
    end
  end

  desc 'Initial Deploy'
  task :initial do
    on roles(:app) do
      before 'deploy:restart', 'puma:start'
      invoke 'deploy'
    end
  end

  desc 'Restart application'
  task :restart do
    on roles(:app), in: :sequence, wait: 5 do
      Rake::Task['puma:restart'].reenable
      invoke 'puma:restart'
    end
  end
  before :starting,  :check_revision
  after  :finishing, :cleanup
end

# puma
set :puma_threads,    [4, 16]
set :puma_workers,    0
set :puma_bind,       "unix://#{shared_path}/tmp/sockets/#{fetch(:application)}-puma.sock"
set :puma_state,      "#{shared_path}/tmp/pids/puma.state"
set :puma_pid,        "#{shared_path}/tmp/pids/puma.pid"
set :puma_access_log, "#{release_path}/log/puma.access.log"
set :puma_error_log,  "#{release_path}/log/puma.error.log"
set :puma_preload_app, true
set :puma_worker_timeout, nil
set :puma_init_active_record, true

namespace :puma do
  desc 'Create Directories for Puma Pids and Socket'
  task :make_dirs do
    on roles(:app) do
      execute "mkdir #{shared_path}/tmp/sockets -p"
      execute "mkdir #{shared_path}/tmp/pids -p"
    end
  end
  before :start, :make_dirs
end
```

config/deploy/production.rb に接続情報を追記

```ruby
# production.rb

server '<server_name>', roles: %i[app web db]
set :user, '<user>'
set :stage, :production
set :ssh_options, forward_agent: true
```

puma を使う為 config.ru を作成

```ruby
# config.ru

require './index'
run Sinatra::Application
```

最後にデプロイしちゃう

```bash
bundle exec cap production deploy:initial
```

注意点として、`secret.yml`とか`database.yml`を.gitignore している場合はデプロイでファイルが配置されないので、手動で配置するかタスクを追加してあげる必要がある。

## Nginx

最後に nginx の設定をする。ちなみに CentOS7 を使ってる。

/etc/nginx/conf.d/<app_name>.conf

```conf
upstream <app_name> {
    server unix:/var/www/<app_name>/shared/tmp/sockets/<app_name>-puma.sock fail_timeout=0;
}

server {
    listen 80;
    server_name <server_name>;
    root /var/www/<app_name>/current/public;

    location / {
      proxy_pass http://<app_name>;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header Host $http_host;
    }

    access_log /var/log/nginx/<app_name>.access.log;
    error_log /var/log/nginx/<app_name>.error.log;

    error_page 500 502 503 504 /500.html;
    client_max_body_size 4G;
    keepalive_timeout 10;
}
```

nginx を restart して完了だ

```bash
sudo systemctl restart nginx
```

`http://<server_name>/v1/books/1` にアクセスすると、DB の Books テーブルから Id:1 のデータを引っ張ってきて、JSON で返してくれる。

## さいごに

振り返るとたいした量じゃないんだけど、これだけやるのにやっぱり半日は掛かってしまった。

考えることなんてほとんど無いんだから、もっとサクサクっと構築できるようにしたい。

## 実行環境

- Ruby 2.3.3
- Sinatra 2.0.3
- activerecord-sqlserver-adapter 5.1.6
- tiny_tds 2.1.2
- puma 3.11.4
- capistrano 3.11.0
