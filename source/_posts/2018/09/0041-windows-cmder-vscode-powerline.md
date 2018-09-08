---
title: 'WindowsでCmderでVSCodeでPowerline'
date: 2018-09-07 11:41:19
tags:
  - vscode
  - powerline
---

唐突に windows で powerline が使いたくなり、powershell に入れようとしたら上手くいかなかった。
しかし cmder では上手く適用できたので、VSCode のターミナルとして使えるようになるまでの手順を残す。

<!-- more -->

# Cmder のインストール

[Cmder | Console Emulator](http://cmder.net/)から Mini 版をダウンロードする。VSCode のターミナルとして使う為、Full 版の充実したコンソールは必要ないという判断。

ダウンロードしたら解凍して適当な場所に配置する。今回は`C:\Program Files\cmder_mini`に配置した。

以下注意点が 2 点

- `C:\Program Files (x86)`に配置しようとしたら、`\cmder_mini\vendor\lib\lib_base"" の使い方が誤っています。`という警告が出た。別の場所がいいのだろう。
- `cmder_mini`ディレクトリをコピーして配置したら起動に管理者権限を要求されたが、移動して配置したらそのまま起動できた。移動して配置しよう。

初回起動でこのような画面が立ち上がる。

![初回起動時の画面](/images/41-01.png)

# Powerline のインストール

[AmrEldib/cmder-powerline-prompt: Custom prompt for Cmder on Windows](https://github.com/AmrEldib/cmder-powerline-prompt)からリポジトリを zip でダウンロードしてくる。（clone でも良いよ）

そして以下のファイルを`C:\Program Files\cmder_mini\config\`へコピーする。

- powerline_core.lua
- powerline_git.lua
- powerline_prompt.lua

再度起動すると、Powerline が適用されているのが分かる。

![Powerline適用後の画面](/images/41-02.png)

ただまだフォントが化けているが、気にせず進める。

## Powerline-font のインストール

Powerline のフォントは色々あるようだが、今回は[powerline/fonts: Patched fonts for Powerline users.](https://github.com/powerline/fonts)を使う。

これも zip でダウンロードし、PowerShell で`.\install.ps1`を実行する。そうするとダウンロードしたフォントが全て Windows にインストールされる。時間が掛かるのでしばし待とう。

# VSCode のターミナルを Cmder にする。

cmder のインストールディレクトリ（`C:\Program Files\cmder_mini`）に以下の内容で`vscode.bat`というバッチを作成する。

```
@echo off
SET CMDER_ROOT=C:\Program Files\cmder_mini
"%CMDER_ROOT%\vendor\init.bat"
```

そして VSCode の`settings.json`に以下設定を追加する。

```json
"terminal.integrated.shell.windows": "C:\\WINDOWS\\sysnative\\cmd.exe",
"terminal.integrated.shellArgs.windows": ["/K", "C:\\Program Files\\cmder_mini\\vscode.bat"]
```

## ターミナルのフォント設定

VSCode の`settings.json`で設定する。

```json
"terminal.integrated.fontFamily": "Source Code Pro for Powerline"
```

先にインストールしたフォントで`**** Powerline`という名前のものならなんでもいいだろう。ただ Cmder のコンソールで表示した時と VSCode のターミナルで表示した時では、同じフォントでも表示がかなり違ったので注意（？）が必要。

![VSCodeのターミナル](/images/41-03.png)

イイ感じになってきた

## カーソルの位置が一文字ずれる問題

プロンプトが`λ`だと、入力の 1 文字目のカーソルと入力位置がずれる問題がある。なのでプロンプトを変えてしまえばいい。

`powerline_core.lua`の 113 行目、`plc_prompt_lambSymbol`を修正することで変えられる。

```lua
if not plc_prompt_lambSymbol then
    plc_prompt_lambSymbol = "λ" -- ここを別の文字に変える
end
```

![プロンプト修正後のターミナル](/images/41-04.png)

OK！

---

ちょっと追記：

プロンプトを `>` に変更したら、node の REPL を実行した時にプロンプトが変わらず分かり辛かった。`>` ではなく `$` のほうがいいかも

---

# 環境

- Windows 10
- VSCode 1.25.1
- cmder_mini 1.3.6

# 参考

- [2017 年お手軽なのに格好良くて最強の Windows ターミナル環境構築。「Windows8.1 に cmder ＋ Powerline」にて - Qiita](https://qiita.com/hmcGit/items/98b80507406cf25c10b4)
- [Windows に Cmder で Bash 環境を作って日本語が表示できるようになるまでの覚え書き - Qiita](https://qiita.com/alchemist/items/f2f54cfe545c84af9430)
- [Integrated Terminal in Visual Studio Code](https://code.visualstudio.com/docs/editor/integrated-terminal)
