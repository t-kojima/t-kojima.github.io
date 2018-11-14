---
title: '[MySQL] Unable to load authentication plugin 'caching_sha2_password'
date: 2018-11-14 10:52:38
categories:
  - MySQL
tags:
  - docker
  - mysql
---

Docker で MySQL を起動して接続しようとしたら`Unable to load authentication plugin 'caching_sha2_password`というエラーに遭遇した。

<!-- more -->

# 動作環境

- Linux Mint 19
- docker 18.06.0-ce
- mysql 8.0.4

# 現象

まずいつものように Docker で MySQL を起動する。

```bash
$ docker run --name mysql -e MYSQL_ROOT_PASSWORD=mysql -p 3306:3306 -d mysql
```

もしくは既にコンテナ作成済みなら以下のように start させる。

```bash
$ docker start mysql
```

すると Docker 上で MySQL が起動する。ここは問題ない。

その後 DB クライアント（今回は[DBeaver](https://dbeaver.io/)を使う）からアクセスしようとすると、以下のエラーになる。

![通信テストでエラー](/images/49-01.png)

# 原因

まあ Docker が原因ではないと思っていたが、MySQL のバージョン 8 からは[デフォルトの認証方式が変わったらしい。](https://yoku0825.blogspot.com/2018/01/mysql-804.html)

MySQL の設定（`mysql.userテーブルのpluginカラム`）を直せば良いようだが、Docker で MySQL コンテナを作成する度に設定を直すのは大変なので、簡単に回避できる策を探す。

……

と、スタックオーバーフローで[それっぽいのを発見した。](https://stackoverflow.com/questions/49019652/not-able-to-connect-to-mysql-docker-from-local)

これによると`--default-authentication-plugin=mysql_native_password`オプションをつけて起動すると良いみたいだ。

一度コンテナを捨てて、下記コマンドで再作成してみる。

```bash
docker run --name mysql -e MYSQL_ROOT_PASSWORD=mysql -p 3306:3306 -d mysql --default-authentication-plugin=mysql_native_password
```

そして再度アクセスすると…

![接続成功](/images/49-2.png)

いけた！

いやあ、久しぶりに使おうとしたらアクセスできなかったので、焦ったぜ。

# 参考

- [日々の覚書: MySQL 8.0.4におけるデフォルト認証形式の変更](https://yoku0825.blogspot.com/2018/01/mysql-804.html)
- [not able to connect to mysql docker from local - Stack Overflow](https://stackoverflow.com/questions/49019652/not-able-to-connect-to-mysql-docker-from-local)
