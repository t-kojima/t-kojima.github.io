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

# VSCode に適用

VSCode の`settings.json`でフォントを設定する。

```json
"terminal.integrated.fontFamily": "Source Code Pro for Powerline"
```

カンタン！

# Powerline-shell の設定

# 環境

- Linux Mint 19
- powerline-shell 0.5.4

# 参考

- [powerline-shell – 美しくて便利なシェルのプロンプト – GitHub じゃ！Python じゃ！](https://githubja.com/b-ryan/powerline-shell)
