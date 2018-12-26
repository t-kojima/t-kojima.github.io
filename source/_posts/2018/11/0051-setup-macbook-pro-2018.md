---
title: 'MacBookPro2018のセットアップ'
date: 2018-12-20 10:09:54
categories:
  - Mac
tags:
  - mac
---

MacBookPro 13インチ TouchBarモデルを手に入れたので、セットアップ手順を残しておく。

<!-- more -->

# 初期設定

開封後、MacBookを開くと勝手に初期セットアップのウィザードが開始される。

最近のOSはなんでもそうだけど、特に難しいことはない。むしろ「Hey!Siri!」が何度も言わされて若干恥ずい。。。

# 環境設定の変更

細かいのが多いので箇条書きで

- システム環境設定 - 共有 からコンピュータ名を変更する。「○○のMacBook」とかになっているので。
- システム環境設定 - セキュリティとプライバシー - ファイアウォール でファイアウォールを有効にする。
- Finder - 環境設定 - 詳細 から「すべてのファイル名拡張子を表示」で拡張子を表示する。
- 日本語入力のライブ変換をOFF

あんま多くなかった

## ディレクトリ表示を英語に変更

デフォルトだとホーム配下のディレクトリ名が日本語（ドキュメントとか）になっているので、英語（documents）に直したい。

10.11 El Capitanからはディレクトリ内にある`.localized`を削除すると良いらしい。
ちょっとやり方を調べたけど、個々に消すしかないっぽい（というか探す時間で消した方が早い）

```bash
rm ~/Downloads/.localized
rm ~/Documents/.localized
rm ~/Pictures/.localized
rm ~/Applications/.localized
rm ~/Music/.localized
rm ~/Public/.localized
rm ~/Desktop/.localized
rm ~/Movies/.localized
rm ~/Library/.localized
```

# パッケージマネージャの導入

MACといえばHomebrew！

Homebrew（ホームブルー）にはビールを自家醸造的な意味があるらしい。自前でビルドするのが似ているのだとか。。。またどうでもいい知識が増えた。

[macOSにHomebrewをインストール - Qiita](https://qiita.com/pypypyo14/items/4bf3b8bd511b6e93c9f9)を参考にインストールする。

## xcode-select

```bash
$ xcode-select --install
xcode-select: note: install requested for command line developer tools
```

ポップアップが表示されるので言われるがままにインストールする。

## homebrew

[公式サイト](https://brew.sh/index_ja.html)のスクリプトを貼り付けろということなので、張り付けて実行する。

```bash
$ /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

ダラララとコンソールが流れ、パスワードを求められたりするので入力したりするとインストールされる。

```bash
$ brew -v
Homebrew 1.8.6
Homebrew/homebrew-core (git revision ba68; last commit 2018-12-19)
```

OKやね

## homebrew cask

どうもGUIツールをHomebrewで入れる場合はcaskなる拡張が必要のようだ。入れよう。

```bash
$ brew tap caskroom/cask
```

たいしてコンソールも流れず、パスワードも求められずにインストールされる。

```bash
$ brew -v
Homebrew 1.8.6
Homebrew/homebrew-core (git revision ba68; last commit 2018-12-19)
Homebrew/homebrew-cask (git revision bca0; last commit 2018-12-19)
```

入った！

# アプリケーションを入れる

brew caskで大体入る

```bash
brew cask install google-chrome
brew cask install visual-studio-code
brew cask install zoomus
brew cask install dropbox
```

# SSHのキーを作成

Githubとかで使うので生成

```bash
ssh-keygen -t rsa -b 4096 -C "hogefuga@example.com"
```

参考

- [お前らのSSH Keysの作り方は間違っている - Qiita](https://qiita.com/suthio/items/2760e4cff0e185fe2db9)

# さいごに

他にもPowerlineとか入れたいけど、とりあえず普段使いに困らないレベルになったのでここまで

## 環境

- Mac OSX 10.14.1 Mojave
