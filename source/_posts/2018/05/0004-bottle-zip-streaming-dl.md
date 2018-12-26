---
title: '[Python] Bottleでzipをストリーミングダウンロード'
date: 2018-05-15 20:47:01
categories:
  - Python
tags:
  - python
  - bottle
  - zipstream
---

## 適当なキーワードでパッケージを探したら本当にあった

<a href="https://github.com/fringd/zipline" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

Ruby に[zipline](https://github.com/fringd/zipline)という gem がある。ファイルのリストを渡すと zip に固めてストリーミングダウンロードしてくれるという Rails 向けの gem だ。今回同じことを Python で行いたく、似たような機能のパッケージ探してみた。

<!-- more -->

すると全く同じ名前のパッケージを見つけたけど、これ[zipline](https://github.com/quantopian/zipline)アルゴリズムトレードライブラリだ！

しょうがないから google で「python zip stream」あたりのキーワードで適当に検索を掛けてみると、ドンピシャなパッケージ[python-zipstream](https://github.com/allanlei/python-zipstream)が一発ヒットしてビビる。
これは使えそうだ

<a href="https://github.com/allanlei/python-zipstream" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

## ダミーファイル作成

まずダウンロードテスト用のダミーファイルを作る

`dd if=/dev/random of=dummy1.txt bs=1M count=100`
`for i in {1..10} ; do dd if=/dev/random of=dummy${i}.txt bs=1M count=100; done`

参考：[Bash でいろいろループする - Qiita](https://qiita.com/ytyng/items/d7afe80ef9da7aa8b721)

これで`dummy1.txt`～`dummy10.txt`まで 100MB のファイルが 10 個作成された。

## zip をストリーミングダウンロード

先に成功したコードを挙げておく。
これで開発環境の場合は`http://localhost:3030/zip`にアクセスすると、`dummy1.txt`～`dummy10.txt`を zip した`files.zip`がストリーミングでダウンロードされる。

```python
import zipstream
from pathlib import Path

@route('/zip')
def zip():
    """ download zip with streaming """
    path = Path('<temp files dir>')

    def generator():
        zip = zipstream.ZipFile(mode='w', compression=zipstream.ZIP_DEFLATED)
        zip.write((path / 'dummy1.txt').as_posix(), arcname='1.txt')
        zip.write((path / 'dummy2.txt').as_posix(), arcname='2.txt')
        zip.write((path / 'dummy3.txt').as_posix(), arcname='3.txt')
        zip.write((path / 'dummy4.txt').as_posix(), arcname='4.txt')
        zip.write((path / 'dummy5.txt').as_posix(), arcname='5.txt')
        zip.write((path / 'dummy6.txt').as_posix(), arcname='6.txt')
        zip.write((path / 'dummy7.txt').as_posix(), arcname='7.txt')
        zip.write((path / 'dummy8.txt').as_posix(), arcname='8.txt')
        zip.write((path / 'dummy9.txt').as_posix(), arcname='9.txt')
        for chunk in zip:
            yield chunk

    zip_filename = 'files.zip'
    response.content_type = 'application/zip'
    response.set_header('Content-Disposition', f'attachment; filename="{zip_filename}"')
    response.body = generator()
    return response
```

本家の`README`を読んだ感じ、使い方がよく理解できなかったので軽く解説を入れてみる。

```python
zip = zipstream.ZipFile(mode='w', compression=zipstream.ZIP_DEFLATED)
```

まずは ZipFile のインスタンスを作る。`zipstream.ZipFile`は`zipfile.ZipFile`を継承したラッパークラスなので、共通のインターフェースになっている。[ここ](https://docs.python.jp/3/library/zipfile.html)を読むと分かりやすいかもしれない。

`mode='w'`を指定しているが、ここはデフォルトも`'w'`だし、`'w'`が含まれないと`RuntimeError`になる。
`compression`はデフォルト`ZIP_STORED`なのでデフォルトだと圧縮されない。取り合えず圧縮したいレベルなら`ZIP_DEFLATED`でお K。

- ZIP_STORED : 圧縮無しでファイルをまとめるだけ
- ZIP_DEFLATED : 通常の ZIP 圧縮、`zlib`モジュールが必要
- ZIP_BZIP2 : BZIP2 での圧縮を行う、`bz2`モジュールが必要
- ZIP_LZMA : LZMA での圧縮を行う、`lzma`モジュールが必要

```python
zip.write((path / 'dummy1.txt').as_posix(), arcname='1.txt')
```

ZipFile インスタンスにファイルを渡していく。ここも`zipfile.ZipFile#write`とインターフェースは同じで、`write('ファイルパス', arcname='アーカイブされるファイル名', compress_type='個別の圧縮指定')`になる。
ここでディレクトリを渡してしまうと**空のディレクトリ**が入ってしまうので注意が必要。

`write_iter(arcname, iterable, compress_type)`メソッドではファイルパスではなくイテレータを渡すこともできるようだ。

`.as_posix()`してるのはファイルパスとして`pathlib`を受け付けないから...残念！

```python
for chunk in zip:
    yield chunk
```

`zipstream.ZipFile`はイテレータで圧縮済みデータを返してくれる。
具体的には`write`したファイルのリストからファイルを順次読み出し、`1024 * 8`バイト読んで圧縮して都度データを返す。

```python
zip_filename = 'files.zip'
response.content_type = 'application/zip'
response.set_header('Content-Disposition', f'attachment; filename="{zip_filename}"')
response.body = generator()
return response
```

最後は`bottle`の`response`部分だ。`Content-Type`は`'application/zip'`を指定する。`Content-Disposition`にはファイルをアタッチすることで即ダウンロードする動きをしてくれる。`response.body`はジェネレータを受けることができるので、`generator()`関数をそのままセットするとストリーム処理するようになってくれる、ありがたい。

## Content-Length は？

`Content-Length`を指定しないとダウンロードの残り時間表示が「**不明**」になってしまうが、圧縮後のサイズが事前に分からないのでここはしょうがない。
試しに`Content-Length`に適当な値を設定してみたら、ダウンロードが強制中断になってしまった。

## 実行環境

- Python 3.6.3
- bottle 0.12.13
- zipstream 1.1.4
