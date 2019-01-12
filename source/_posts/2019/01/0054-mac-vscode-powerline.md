---
title: 'MacでVSCodeでPowerline'
date: 2019-01-12 08:45:48
categories:
  - VSCode
tags:
  - vscode
  - powerline
---

Mac にも powerline を入れたい！

過去に Windows と Linux ではやったので、今度は Mac に入れてみる。

- [Windows で Cmder で VSCode で Powerline](https://t-kojima.github.io/2018/09/24/0041-windows-cmder-vscode-powerline/)
- [Linux で VSCode で Powerline](https://t-kojima.github.io/2018/09/26/0042-linux-vscode-powerline/)

<!-- more -->

[Powerline 公式の方法](https://powerline.readthedocs.io/en/2.4/installation/osx.html)でやってみる。

# pip をインストール

まずインストールには Python と pip が必要になるので、バージョンを確認してみる。

```bash
$ python --version
Python 2.7.10
$ pip --version
bash: pip: command not found
```

なんてこった pip がない

というか python 環境が全くない（3 系とか）ので、pyenv から 3 系を入れてみよう。pyenv からインストールすれば pip も一緒にインストールされる。

## pyenv のインストール

[MacOS と Homebrew と pyenv で快適 python 環境を。 - Qiita](https://qiita.com/crankcube@github/items/15f06b32ec56736fc43a) を参考に入れる。

```bash
$ brew install pyenv
==> Installing dependencies for pyenv: autoconf, openssl, pkg-config and readline
==> Installing pyenv dependency: autoconf
==> Downloading https://homebrew.bintray.com/bottles/autoconf-2.69.mojave.bottle.4.tar.gz
######################################################################## 100.0%
==> Pouring autoconf-2.69.mojave.bottle.4.tar.gz
==> Caveats
Emacs Lisp files have been installed to:
  /usr/local/share/emacs/site-lisp/autoconf
==> Summary
🍺  /usr/local/Cellar/autoconf/2.69: 71 files, 3.0MB
==> Installing pyenv dependency: openssl
==> Downloading https://homebrew.bintray.com/bottles/openssl-1.0.2q.mojave.bottle.tar.gz
######################################################################## 100.0%
~~ 省略 ~~

$ pyenv --version
pyenv 1.2.9
```

1.2.9 が入った。

.bash_profile に追記してパスを通す

```bash
$ echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bash_profile
$ echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bash_profile
$ echo 'eval "$(pyenv init -)"' >> ~/.bash_profile
$ source ~/.bash_profile
```

`pyenv install --list` でインストール可能な一覧が見れる。今回は`3.7.2`を入れてみる。

```bash
$ pyenv install 3.7.2
```

入ったらグローバルに設定して完了だ

```bash
$ pyenv global 3.7.2
$ pyenv versions
  system
* 3.7.2 (set by /Users/********/.pyenv/version)
$ python --version
Python 3.7.2
$ pip --version
pip 18.1 from /Users/********/.pyenv/versions/3.7.2/lib/python3.7/site-packages/pip (python 3.7)
```

pip も無事使えるようになっている。

### zlib エラー

macos Mojave ではビルド時に以下のエラーになってしまう。

```bash
zipimport.ZipImportError: can't decompress data; zlib not available
make: *** [install] Error 1
```

参考：[[MacOS Mojave]pyenv で python のインストールが zlib エラーで失敗した時の対応 - Qiita](https://qiita.com/zreactor/items/c3fd04417e0d61af0afe)

以下のコマンドで Mojave 用の macOS SDK header を入れる

```bash
sudo installer -pkg /Library/Developer/CommandLineTools/Packages/macOS_SDK_headers_for_macOS_10.14.pkg -target /
```

再度 `pyenv install 3.7.2` でインストールして成功すれば OK だ

# powerline のインストール

前置き（？）が長くなってしまったが、powerline を入れる。
改めて[Powerline 公式の方法](https://powerline.readthedocs.io/en/2.4/installation/osx.html)でやろう。

[Powerline 導入例 - Qiita](https://qiita.com/tkhr/items/8cc17c02dea1803be9c6)も参考にしながらやる。

```bash
$ pip install powerline-status
Collecting powerline-status
  Using cached https://files.pythonhosted.org/packages/9c/30/8bd3c62642778af9ad813a526c6ff7dd2f98144d6580ad6fab94ca389265/powerline-status-2.7.tar.gz
Installing collected packages: powerline-status
  Running setup.py install for powerline-status ... done
Successfully installed powerline-status-2.7
```

即入った。簡単

ちなみに公式だと`--user`オプションをつけるようあるが、python を homebrew からインストールした際は付けないよう注意書きがしてある。
`pyenv`から入れた場合もオプションをつけたら動かなかった。`pyenv`から入れた場合も同様にオプション無しが正解のようだ。

`powerline-daemon -h` でヘルプが表示されれば OK

## bash に適用

[Shell prompts — Powerline beta documentation](https://powerline.readthedocs.io/en/2.4/usage/shell-prompts.html#bash-prompt)を参考に shell に powerline 適用する。

まずリポジトリのルートを確認

```bash
$ pip --version
pip 18.1 from /Users/********/.pyenv/versions/3.7.2/lib/python3.7/site-packages/pip (python 3.7)
```

ここの`/Users/********/.pyenv/versions/3.7.2/lib/python3.7/site-packages`がルートになる。

```bash
$ echo 'powerline-daemon -q' >> ~/.bash_profile
$ echo '. {repository_root}/powerline/bindings/bash/powerline.sh' >> ~/.bash_profile
$ source ~/.bash_profile
```

これで bash を再起動すると shell に powerline が適用される。

しかしこのままだと文字化けしてしまうので、powerline 用のフォントを入れる。

## フォントをインストール

参考元のままやる

```bash
cd ~/Desktop
git clone git@github.com:powerline/fonts.git
./fonts/install.sh
```

このように powerline 用のフォントが追加された。

![FontBook.app](/images/54-01.png)

### VSCode のフォントを変更

VSCode でユーザー設定を開き、フォントを変更する。

ついでのサイズを 12 にする。ちょうどいいと思う。

```json
    "terminal.integrated.fontFamily": "Source Code Pro for Powerline",
    "terminal.integrated.fontSize": 12,
```

これで VSCode のターミナルに powerline が適用された！

![Powerlineが適用されたターミナル](/images/54-02.png)

# Configuration

デフォルト表記だと git のブランチが表示されなかったりするので、設定を変更する。

参考：[Configuration and customization — Powerline beta documentation](https://powerline.readthedocs.io/en/2.4/configuration.html)

デフォルトの設定はリポジトリルートの奥底にあるが、`~/.config/powerline`配下に設定ファイルを置くと、記述した分だけデフォルト設定を上書きしてくれる。

```bash
mkdir ~/.config/powerline
touch ~/.config/powerline/config.json
code ~/.config/powerline/config.json
```

VSCode で編集

```json
{
  "ext": {
    "shell": {
      "theme": "custom"
    }
  }
}
```

themes/custom.json も作る

```bash
touch ~/.config/powerline/themes/shell/custom.json
code ~/.config/powerline/themes/shell/custom.json
```

こちらも VSCode で編集

```json
{
  "segments": {
    "left": [
      {
        "function": "powerline.segments.shell.mode"
      },
      {
        "function": "powerline.segments.common.net.hostname",
        "priority": 10
      },
      {
        "function": "powerline.segments.common.env.user",
        "priority": 30
      },
      {
        "function": "powerline.segments.common.env.virtualenv",
        "priority": 50
      },
      {
        "function": "powerline.segments.common.vcs.branch",
        "priority": 40,
        "args": {
          "status_colors": true
        }
      },
      {
        "function": "powerline.segments.shell.cwd",
        "priority": 10,
        "args": {
          "dir_limit_depth": 1
        }
      },
      {
        "function": "powerline.segments.shell.jobnum",
        "priority": 20
      },
      {
        "function": "powerline.segments.shell.last_pipe_status",
        "priority": 10
      }
    ]
  }
}
```

`powerline-daemon --refresh` すると適用される。

![コンフィグを修正した後のターミナル](/images/54-03.png)

ここまでがんばったが、設定のやり方が分かり辛すぎる...
powerline の issue で 「documentation is confusing」 とか言われてて笑える w...

# さいごに

結構めんどかった。インストール方法を調べると色々な方法が出てきてどれが適当かよくわからない。

今回公式の方法に沿って行ったが、Linux の時と同様に`powerline-shell`を使ったほうがサクっといけたような気がしてならない。

...ま、いいか

## 環境

- Mac OSX 10.14.1 Mojave
- VSCode 1.30.2
- powerline-status 2.7
