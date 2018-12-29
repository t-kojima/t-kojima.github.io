---
title: '停電とNASのシャットダウン'
date: 2018-12-29 08:40:03
categories:
tags:
  - qnap
---

今朝停電があって、NAS（QNAP）のシャットダウンでやっちまった....

<!-- more -->

## 何があったか

今朝の7時前辺りから停電があり、NASの電源がUPSに切り替わっていた。

しかし僕はそれに30分？ほどしてから気付き、慌ててシャットダウンしたが
なんとシャットダウンシーケンス中にUPSが切れてしまった。

丁度スケジュールバックアップとカチあったのも悪かったかもしれない。

### QNAP TS-451P

NASは[QNAPのTS-451P](https://www.qnap.com/ja-jp/product/ts-451+)を使っている。

<a href="https://www.qnap.com/ja-jp/product/ts-451+" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

普段使う性能的には非常に満足していたが、電源周りにちょっと不安があった。具体的には起動・シャットダウンにやたらと時間がかかったり、リブート中にフリーズする...などだ。

### OMRON BY50S

UPSは[OMRONのBY50S](https://www.oss.omron.co.jp/ups/product/by35-120s/by35-120s.html)を使ってる。
元々PC用として正弦波のUPSが必要で使っていたが、後にNAS用に流用していた。

<a href="https://www.oss.omron.co.jp/ups/product/by35-120s/by35-120s.html" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

## その後

停電が復旧し、QNAPを立ち上げたところ正常に立ち上がり、HDDにも障害は無いようだ。

しかしUPSを繋げておきながら、自動シャットダウンの設定をしていなかったのは大いに反省。

## 自動シャットダウンの設定

なので今のうちにやってしまう。鉄は熱いうちに打てというやつだ。

とはいえ、USBケーブルでUPSとNASを繋ぐだけでOKだった。もっと面倒臭い設定だったり、NASにUPS用のソフトをインストールしたり、そもそもNASで使えるソフト何てないんじゃね？とか思っててやっていなかった。

接続するとコントロールパネルでUPSを認識していることを確認することができる。

![コントロールパネル - 外部デバイス - UPS（上部）](/images/52-01.png)

![コントロールパネル - 外部デバイス - UPS（下部）](/images/52-02.png)

これで次回からはOKな...はず

電源抜いてテストでもしてみようかな
