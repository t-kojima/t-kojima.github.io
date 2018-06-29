---
title: 'VirtualBoxとVagrantを試す'
date: 2018-05-15 21:46:21
tags:
- vagrant
- virtualbox
---

Vagrant というものを使ってみたく、Windows10 で VirtualBox+Vagrant の環境を構築してみる。

<!-- more -->

## install

### Vagrant

<a href="https://www.vagrantup.com/downloads.html" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

特に問題なく、ふつーにインストール

### VirtualBox

<a href="https://www.virtualbox.org/wiki/Downloads" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

5.2.10 は技術的な理由で今は使えないとのこと、5.2.8 をダウンロードしインストールする。

<a href="https://www.virtualbox.org/wiki/Download_Old_Builds_5_2" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

## CentOS7 を動かす

```bash
vagrant init geerlingguy/centos7; vagrant up --provider virtualbox
```

vagrant で CentOS7 を動かしてみる。

```bash
Stderr: VBoxManage.exe: error: VT-x is not available (VERR_VMX_NO_VMX)
VBoxManage.exe: error: Details: code E_FAIL (0x80004005), component ConsoleWrap, interface IConsole
```

エラーだ…`VT-x is not available`？はて、BIOS の設定は有効になっているはず…

<a href="http://mochalog.hatenablog.com/entry/2015/12/04/143827" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

<a href="http://www.cyamax.com/entry/2017/05/07/060000" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

うおおお`Hyper-v`動いてるとダメなんかい…

`Hyper-v`自体にこだわりは無いけど、`Docker for Windows`が依存してるので止められない。

しかたないので`virtualbox`は諦めて`Hyper-v`で動かす方向でがんばる

<a href="https://qiita.com/nibral/items/94de6b9787e2aface2aa" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left" data-card-description="0"></a>

```bash
vagrant init geerlingguy/centos7; vagrant up --provider hyperv
```

```bash
The box you're attempting to add doesn't support the provider
you requested. Please find an alternate box or use an alternate
provider. Double-check your requested provider to verify you didn't
simply misspell it.

> お使いの Box は Hyper-v をサポートしておりません。。。
```

[https://app.vagrantup.com/boxes/search](https://app.vagrantup.com/boxes/search)で provider を hyperv にして再度検索

[https://app.vagrantup.com/generic/boxes/centos7](https://app.vagrantup.com/generic/boxes/centos7)に決めた！

```bash
vagrant init generic/centos7; vagrant up --provider hyperv
```

```bash
Script: import_vm_vmcx.ps1
Error:

Hyper-V\Compare-VM : 仮想マシンをインポートできませんでした。
発生場所 C:\Program Files (x86)\HashiCorp\Vagrant\embedded\gems\2.0.4\gems\vagrant-2.0.4\plugins\providers\hyperv\scripts\import_vm_vmcx.ps1:33 文字:14
+ $vmConfig = (Hyper-V\Compare-VM -Copy -GenerateNewID @VmProperties)
+              ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : NotSpecified: (:) [Compare-VM], VirtualizationException
    + FullyQualifiedErrorId : Unspecified,Microsoft.HyperV.PowerShell.Commands.CompareVM
```

Hyper-v のコンソールから全然関係ない仮想マシンをインポートしようとしたらこれもコケる。どうも Hyper-v の「インポート」そのものが死んでる模様...？

心当たりはある。

手持ちの PC と Windows10 の相性が悪く`Fall Creaters Update`を適用していないのだ。（以前適用したらエクスプローラが死ぬという致命的な障害が発生して解決できなかった為…）

試しに box のバージョンを下げてみる

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "generic/centos7"
  config.vm.box_version = "1.3"
end
```

```bash
Script: import_vm_vmcx.ps1
Error:

Hyper-V\Connect-VMNetworkAdapter : Hyper-V で、"螟夜Κ繝阪ャ繝医Ρ繝ｼ繧ｯ" という名前の仮想スイッチが見つかりませんでした。
発生場所 C:\Program Files (x86)\HashiCorp\Vagrant\embedded\gems\2.0.4\gems\vagrant-2.0.4\plugins\providers\hyperv\scripts\import_vm_vmcx.ps1:96 文字:1
+ Hyper-V\Connect-VMNetworkAdapter -VMNetworkAdapter $vmNetworkAdapter  ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidArgument: (:) [Connect-VMNetworkAdapter]、VirtualizationException
    + FullyQualifiedErrorId : InvalidParameter,Microsoft.HyperV.PowerShell.Commands.ConnectVMNetworkAdapter
```

エラーが変わった！[これ](https://qiita.com/euledge/items/fff2659093a13a1e888f)だ！

これはリンク先の「既定のスイッチ」の問題ではなく、単に自分が日本語で仮想スイッチを作成していただけであった。（Hyper-v のバージョンが低いから「既定のスイッチ」がそもそも無い）

仮想スイッチ名をアルファベットのみに修正して再度実行

```bash
    default: Inserting generated public key within guest...
    default: Removing insecure key from the guest if it's present...
    default: Key inserted! Disconnecting and reconnecting using new SSH key...
==> default: Machine booted and ready!
```

起動できた！

## ほんとうは…

`Ansible`のハンズオン環境を作りたかっただけなんだ

こういうのを`yak shaving （ヤクの毛刈り）`って言うんだろうね、実感してしまった

## 実行環境

- Windows10 (バージョン 1607 ビルド 14393.1884)
- Vagrant 2.0.4
