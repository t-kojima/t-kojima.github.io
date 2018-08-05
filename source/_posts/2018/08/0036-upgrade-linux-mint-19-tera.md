---
title: '[Linux] SSD換装ついでに Mint 18 を 19 にアップグレードした'
date: 2018-08-05 23:53:03
tags:
- linux
---

ノート PC を HDD から SSD に換装したので、ついでに OS を Linux Mint 18 から 19 にバージョンアップさせた。

<a href="https://www.amazon.co.jp/dp/B01N3380LJ/ref=twister_B06XW5TV6Y?_encoding=UTF8&psc=1" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

SSD は本当安くなったね…120GB だけど 4000 円とは…

<!-- more -->

で、色々再インストールが必要になるわけだけど

基本的なところは以下のサイトを参考に初期設定を済ませる。

<a href="http://baker-street.jugem.jp/?eid=107" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

つまずくところがあれば追記しようと思ったけど、つまずくところナイ。

あとは開発環境あれこれを整えていく

# VSCode

インストールする方法はいくつかあるけど、Mint の `ソフトウェアの管理` からインストールしてはいけない。

- sudo が command not found になる
- 拡張機能の command が呼べない（extension `command` not found）
- デフォルトターミナルが sh
- 組み込みの Git が使われる

など、若干変な挙動を取る

[公式の.deb](https://code.visualstudio.com/download) からインストールすべき。

# Node

まず Node 自身のバージョン切り替えを行えるよう nvm をインストールする。基本的には[Github の通り](https://github.com/creationix/nvm#installation)にインストールすれば問題ない。

```bash
wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.33.11/install.sh | bash
```

環境変数の export も一緒に.bashrc に書き込んでくれる。楽ちん。

`nvm ls-remote` コマンドでインストール可能なバージョン一覧が表示される。けど、LTS をインストールするなら以下のコマンドで OK

```bash
nvm install --lts
```

今だと v8.11.3 がインストールされる。

## 参考

- [node.js は nvm を利用して install する](https://mosapride.com/index.php/2017/04/21/post-265/)
- [creationix/nvm: Node Version Manager - Simple bash script to manage multiple active node.js versions](https://github.com/creationix/nvm)

# Python

とりあえずデフォルトでは 2.7.15 が入っているらしい

```bash
$ python --version
Python 2.7.15rc1
```

## Pyenv をインストール

まず pyenv をクローンし、環境変数を整える

```bash
git clone https://github.com/yyuu/pyenv.git ~/.pyenv
echo 'export PYENV_ROOT=$HOME/.pyenv' >> ~/.bashrc
echo 'export PATH=$PYENV_ROOT/bin:$PATH' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc
source ~/.bashrc
```

特に問題なく使えるようになる

```bash
$ pyenv --version
pyenv 1.2.6
```

## Python をインストール

続いて最新の python を入れる。

```bash
pyenv install --list
pyenv install 3.7.0
```

するとモジュールエラーとなる。

```bash
ModuleNotFoundError: No module named '_ctypes'
Makefile:1122: recipe for target 'install' failed
make: *** [install] Error 1
```

パッケージが足りなかったようだ。ので追加する。

```bash
sudo apt-get install libffi-dev
```

再度インストールを実行すると、新たなエラーが…

```bash
Installing Python-3.7.0...
WARNING: The Python bz2 extension was not compiled. Missing the bzip2 lib?
WARNING: The Python readline extension was not compiled. Missing the GNU readline lib?
ERROR: The Python ssl extension was not compiled. Missing the OpenSSL lib?

Please consult to the Wiki page to fix the problem.
https://github.com/pyenv/pyenv/wiki/Common-build-problems


BUILD FAILED (LinuxMint 19 using python-build 1.2.6)

Inspect or clean up the working tree at /tmp/python-build.20180805205450.2376
Results logged to /tmp/python-build.20180805205450.2376.log

Last 10 log lines:
                install|*) ensurepip="" ;; \
        esac; \
         ./python -E -m ensurepip \
                $ensurepip --root=/ ; \
fi
Looking in links: /tmp/tmp9w6bwrpv
Collecting setuptools
Collecting pip
Installing collected packages: setuptools, pip
Successfully installed pip-10.0.1 setuptools-39.0.1
```

まだパッケージ諸々が足りていなかったので、追加する。

```bash
sudo apt-get install libssl-dev
sudo apt-get install libbz2-dev libreadline-dev libsqlite3-dev
```

改めて python をインストールする。

```bash
pyenv install 3.7.0
```

今度は OK、global に設定し、pipenv をインストールする。

```bash
pyenv global 3.7.0
pip install pipenv
echo 'export PIPENV_VENV_IN_PROJECT="true"' >> ~/.bashrc
```

`PIPENV_VIEW_IN_PROJECT` 環境変数も.bashrc に追加しておく。

とりあえずこれで完了。細かいパッケージはプロジェクト毎に pipenv で入れればいい。

## 参考

- [Ubuntu で Python の開発環境を整える - Qiita](https://qiita.com/uryyyyyyy/items/268f8dc0d6ec3d7da7e3)
- [Ubuntu 15.10 で pyenv install 3.5.0 したら pip がインストールできなくて落ちる - Qiita](https://qiita.com/utgwkk/items/bf282ca95f64ef7dd594)

# Ruby

Ruby もデフォルトで 2.5.1p57 がインストールされているようだ

```bash
$ ruby -v
ruby 2.5.1p57 (2018-03-29 revision 63029) [x86_64-linux-gnu]
```

## rbenv をインストール

```bash
sudo apt-get install git build-essential libssl-dev
git clone https://github.com/sstephenson/rbenv.git ~/.rbenv
git clone https://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
```

クローンできたら、以下を.bashrc へ追加する

```bash
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
t-kojima@mint2:~$ echo 'eval "$(rbenv init -)"' >> ~/.bashrc
source ~/.bashrc
```

## Ruby をインストール

`rbenv install --list` でインストール可能なバージョンを確認する。今回は 2.5.1 にする。

```bash
rbenv install 2.5.1
```

ちょっと時間が掛かるけど…完了したら rehash する。

```bash
rbenv rehash
```

global を 2.5.1 に設定して、完了。

```bash
$ rbenv global 2.5.1
$ ruby -v
ruby 2.5.1p57 (2018-03-29 revision 63029) [x86_64-linux]
```

## bundler をインストール

```bash
gem install bundler
```

## 参考

- [Ubuntu14.04 に rbenv をインストールして Ruby のバージョン管理 - Qiita](https://qiita.com/kazoo04/items/289c4f6a8501df3a7359)

# Docker

```bash
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu xenial stable"
sudo apt-get update
sudo apt-get install docker-ce
```

## MySQL を入れる

とりあえず MySQL を入れる

```bash
sudo docker run --name mysql -e MYSQL_ROOT_PASSWORD=mysql -p 3306:3306 -d mysql
```

## 参考

- [Get Docker CE for Ubuntu | Docker Documentation](https://docs.docker.com/install/linux/docker-ce/ubuntu/)
- [Install docker CE on Linux Mint 18.3](https://gist.github.com/dweldon/39ca0536168a92815a56df44eb55)
