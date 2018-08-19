---
title: '[VSCode] Pythonデバッグの挙動 - 2018.3.0 (28 Mar 2018)'
date: 2018-05-10 00:41:44
tags:
  - vscode
  - python
---

## 突然デバッグの挙動が変わった

VSCode で Python のデバッグをしようとした時、**デバッグコンソール**ではなく**ターミナル**に`Python Debug Console`が立ち上がってきた。デバッグ自体はいつも通りできるけど…なんだか気持ち悪い。

<!-- more -->

![エラーメッセージ](/images/03-01.png)

デバッグを停止した時コンソールは自動的に閉じないので、連続でデバッグを繰り返していると`Python Debug Console`がターミナルにどんどん開かれていく。この時新しい`Python Debug Console`を開いても、ブレークポイントが古い`Python Debug Console`に紐付けられたままだったりして、思い通りの挙動をしないことがしばしばあってあまりよろしくない感じだった。

- 後になって分かったが、デバッグを開始した時新しい`Python Debug Console`が開かれる場合と、既存のコンソールを再利用する場合の両方の挙動が確認できた。デバッグを開始するタイミングなのだろうか？
- ユーザー設定に`"debug.internalConsoleOptions"や"debug.openDebug"`というパラメータがあるが、これらを変更しても挙動の変化はなかった。`"neverOpen"`にしても問答無用で開くし…

## PythonExtension のアップデートが原因

> 2018.3.0 (28 Mar 2018)
>
> Enhancements
>
> 10 . When debugging, use Integrated Terminal as the default console. (#526)

3/28 のアップデートでデバッグのデフォルトコンソールが`Integrated Terminal`に変更になっていた。
元の[Issie#526](https://github.com/Microsoft/vscode-python/issues/526)を確認すると、既存のデバッグコンソールでは入力が必要なアプリのデバッグができないとのこと。
なるほど、確かに

## 元に戻す

とはいえ元のデバッグコンソールのほうがやりやすかったので戻してみる。`launch.json`を修正し、デバッグの構成で利用するコンソールを指定してあげればいい。

```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: Current File",
            "type": "python",
            "request": "launch",
            "program": "${file}",
            "console": "none"
        }
    ]
}
```

`"console":"none"`を追記することで、既存のデバッグコンソールでデバッグが動くようになる。
やったね！
