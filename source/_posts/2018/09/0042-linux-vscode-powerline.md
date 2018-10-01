---
title: 'LinuxでVSCodeでPowerline'
date: 2018-09-26 14:53:39
tags:
  - vscode
  - powerline
---

Linux にも powerline を入れたい。Windows ほど面倒ではなくサクっとできるのだけど、次回クリーンインストール時に多分手順を覚えていないので残しておく。

ちなみに Linux は Mint 19 "Tera" です。

<!-- more -->

# Powerline-shell

<a href="https://github.com/b-ryan/powerline-shell" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

powerline-shell を使ってシェルに powerline を適用させる。Python で作られていて pip で簡単にインストールできる。

```bash
pip install powerline-shell
```

bash を使っている場合は、.bashrc に以下を追記

```bash
function _update_ps1() {
    PS1=$(powerline-shell $?)
}

if [[ $TERM != linux && ! $PROMPT_COMMAND =~ _update_ps1 ]]; then
    PROMPT_COMMAND="_update_ps1; $PROMPT_COMMAND"
fi
```

bash 以外は[README の Setup](https://github.com/b-ryan/powerline-shell#setup)を参考にして欲しい。zsh とか fish とかある。

これで powerline がシェルに適用されるが、まだフォントが無いので記号が表示されない。

続けて powerline 用のフォントをインストールしていく。

# Powerline-fonts

<a href="https://github.com/powerline/fonts" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

[Quick Installation](https://github.com/powerline/fonts#quick-installation)に従ってやればいい、クローンしてシェル叩くだけだね。

```bash
git clone https://github.com/powerline/fonts.git --depth=1

cd fonts
./install.sh

cd ..
rm -rf fonts
```

これでフォントが追加された。

# VSCode に適用

VSCode の`settings.json`でフォントを設定する。

```json
"terminal.integrated.fontFamily": "Source Code Pro for Powerline"
```

カンタン！

ちょっとフォントの大きさが気になる場合は調整すると良い。デフォルトが 14 なので少し小さめの 12 あたりにしてみる。

```json
"terminal.integrated.fontSize": 12
```

以下のように VSCode のターミナルに Powerline が適用できた。

![Powerline適用後の画面](/images/42-01.png)

# Powerline-shell の設定

Powerline の適用はできたが、プロンプトにカレントのパスがズラズラ表示されるのは若干イケてない。なので設定を変更してみる。

powerline-shell では設定ファイルを生成し、カスタマイズすることができる。（設定ファイルを生成していない状態では、デフォルトの設定が適用される）

```bash
powerline-shell --generate-config > ~/.powerline-shell.json
```

これで`.powerline-shell.json`が生成されるので、それを編集する。

今回はディレクトリを展開せずに現在のディレクトリのみを表示させたいので、以下のオプションを追加する。

```json
  "cwd": {
    "mode": "dironly"
  }
```

ちなみに json 全体は以下の通りだ。

```json
{
  "segments": [
    "virtual_env",
    "username",
    "ssh",
    "cwd",
    "git",
    "hg",
    "jobs",
    "root",
    "read_only"
  ],
  "cwd": {
    "mode": "dironly"
  },
  "vcs": {
    "show_symbol": "true"
  }
}
```

すると以下のように現在のディレクトリしか表示されなくなった。

![設定変更後の画面](/images/42-02.png)

# 環境

- Linux Mint 19
- powerline-shell 0.5.4

# 参考

- [powerline-shell – 美しくて便利なシェルのプロンプト – GitHub じゃ！Python じゃ！](https://githubja.com/b-ryan/powerline-shell)
