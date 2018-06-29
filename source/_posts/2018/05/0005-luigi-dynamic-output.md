---
title: '[Python] luigiで動的アウトプット'
date: 2018-05-15 20:47:52
tags:
- python
- luigi
---

<a href="https://github.com/spotify/luigi" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

最近[luigi](https://github.com/spotify/luigi)というワークフローフレームワークで Python のバッチ処理を書いている。

この luigi の特徴として複数のタスクで依存関係を持たせることができ、以下のような流れで依存関係を解決する。

<!-- more -->

- タスクは完了すると`output`としてファイルを出力する。
- タスクは起動の条件となる前提タスク（`requires`）がある場合、前提タスクが先に走る。
- 前提タスクの`output`が既に存在する場合は、前提タスクが走らずにタスクが起動する。

つまり`output`のファイルでタスクの動きを制御しているのだ。

## 今回やりたかったこと

今回やりたかったことは以下 3 点

- ターゲットフォルダからファイルを 1 個自動で選ぶ
- バックアップフォルダへコピーする
- コピー先のパスを DB に登録する

これだけ、内容自体はよくある処理で、個別に実装する分にはイージーだ。

それぞれを`Pickup`,`FileCopy`,`Archive`という名前でタスクを作成する。

- ターゲットフォルダからファイルを 1 個自動で選ぶ -> `Pickup`
- バックアップフォルダへコピーする -> `FileCopy`
- コピー先のパスを DB に登録する -> `Archive`

処理の流れは`Pickup`->`FileCopy`->`Archive`なんだけど、依存関係を遡って処理していくので最初に動かすタスクは`Archive`になる。

こんな感じで流れる

1. `Archive`の`output`を確認、ファイルがあれば何もせず終了、無ければ次へ
2. `Archive`の`requires`を確認、あれば依存タスク（`FileCopy`）へ
3. `FileCopy`の`output`を確認、ファイルがあれば`Archive`だけ`run`して終了、無ければ次へ
4. `FileCopy`の`requires`を確認、あれば依存タスク（`Pickup`）へ
5. `Pickup`の`output`を確認、ファイルがあれば`FileCopy`->`Archive`の順で`run`して終了、無ければ次へ
6. `Pickup`の`requires`を確認、依存タスクが無ければ`Pickup`->`FileCopy`->`Archive`の順で`run`して終了

コードはこんな感じになるだろう

```python
#!/usr/bin/env python
# coding: utf-8

import luigi
import shutil


class Pickup(luigi.Task):

    def run(self):
        # ファイルピックアップ処理
        with self.output().open('w') as f:
            f.write('<ピックアップ結果のファイルパス>')

    def output(self):
        return luigi.LocalTarget('pickup-output-file')


class FileCopy(luigi.Task):

    def requires(self):
        return Pickup()

    def run(self):
        with self.input().open('r') as f:
            input_path = f.read()
        shutil.copy2(input_path, '<アウトプットパス>')

    def output(self):
        return luigi.LocalTarget('<アウトプットパス>')


class Archive(luigi.Task):

    def requires(self):
        return FileCopy()

    def run(self):
        # ファイルアーカイブ処理
        with self.output().open('w') as f:
            f.write('Archive done')

    def output(self):
        return luigi.LocalTarget('archive-output-file')
```

## output をどうするか

上記のコードで`output`で出力するファイルをどうするか考える

`Archive`と`Pickup`のようにファイル名が決まっている場合（`requires`で呼ばれたタイミングでは決まっている場合）は問題ないが、`FileCopy`のように動的にファイルを決めたい場合（`requires`時点ではまだファイル名が決定できない場合）は途端に難しくなる。

上記のコードでは`Archive`から依存タスクを遡っていって、実際は`Pickup`が最初に`run`される。この時に処理するファイルが**自動**でピックアップされる（っていう仕様）ので、ここで初めてコピー・DB 登録されるファイルが決まるのである。
しかし luigi では依存タスクの解決をする時点でファイル名は定まっていなければならない。`Archive`が`FileCopy`に依存しているとチェックする一番最初の箇所で、`FileCopy`の`output`は何というファイルなのかを求められてしまうのだ。

処理されるファイル名が動的に決定するというパターンは決してレアケースではなく、むしろありふれた処理だと思う。しかし luigi ではタスクの依存性を解決していく中でファイル名を動的に取得するという動きができない。

依存性を解決する為にファイル名が必要、ファイル名を決める為には依存性の解決が必要、ルイージのジレンマだ。

## ではどうするの？

そもそも`requires`と`output`で連鎖的に依存関係を解決していく方法では、`output`の出力ファイルは`requires`時点で決定していなければならない」という制約があることを理解することだ。
今回のように動的に`output`を定めたい場合は`requires`を用いらずに動的に依存関係を解決する方法が luigi から提供されている。

### Dynamic dependencies を使う

[Dynamic dependencies（動的依存関係）](http://luigi.readthedocs.io/en/stable/tasks.html#dynamic-dependencies)を使うと`run`した際に動的に依存関係を解決できる。

[このサンプルコード](https://github.com/spotify/luigi/blob/master/examples/dynamic_requirements.py)も併せて確認するとよい

書き換えたコードが以下のものだ

```python
#!/usr/bin/env python
# coding: utf-8

import luigi
import shutil


class Pickup(luigi.Task):

    def run(self):
        # ファイルピックアップ処理
        with self.output().open('w') as f:
            f.write('<ピックアップ結果のファイルパス>')

    def output(self):
        return luigi.LocalTarget('pickup-output-file')


class FileCopy(luigi.Task):
    input_path = luigi.Parameter()

    def run(self):
        shutil.copy2(self.input_path, '<アウトプットパス>')

    def output(self):
        return luigi.LocalTarget('<アウトプットパス>')


class Archive(luigi.Task):

    def run(self):

        pickup = Pickup()
        yield pickup
        with pickup.output().open('r') as f:
            input_path = f.read()

        filecopy = FileCopy(input_path)
        yield filecopy
        output_path = filecopy.output()

        # ファイルアーカイブ処理

        with self.output().open('w') as f:
            f.write('Archive done')

    def output(self):
        return luigi.LocalTarget('archive-output-file')
```

`requires`が全てのタスクから消えて、依存関係を示す処理が`Archive`の`run`に取り込まれていることがわかる。

このタスクを実施すると以下の流れで実行される。

1. `Archive`の`output`を確認、ファイルがあれば何もせず終了、無ければ次へ
2. `Archive`の`run`を実行
3. `Pickup`タスクを作成
4. `Pickup`の`output`を確認、ファイルが無ければ`run`を実行（`yield pickup`部分）
5. `Pickup`の`output`を引数として、`FileCopy`タスクを作成
6. `FileCopy`の`output`を確認、ファイルが無ければ`run`を実行（`yield filecopy`部分）
7. `FileCopy`の`output`を引数として、ファイルアーカイブ処理を実施
8. `Archive`の`output`を出力して終了

`Archive`の`run`がちょっとごちゃっとなってしまったけど、目的は達成できた！

ただ`Archive`からみた依存関係は`Pickup`も`FileCopy`も同じレベルに変わったことは注意したい。`Pickup`と`FileCopy`の間に依存関係はないのだ。
だから`FileCopy`の`output`が存在していても、無関係に`Pickup`は毎回実行される。
まあ、`FileCopy`は動的に処理したいのだから`Pickup`に依存してなくていい、望んでいた形ではある。

## 残課題

- `Archive`は DB 登録する処理だから、`output`は[MySqlTarget](http://luigi.readthedocs.io/en/stable/api/luigi.contrib.mysqldb.html)にしたほうがいい
- `Pickup`は毎回無条件に実施するのであればタスクでなくてよかったかも？以下のコードはその場で処理してしまっていいかもしれない

```python
        pickup = Pickup()
        yield pickup
        with pickup.output().open('r') as f:
            input_path = f.read()
```

## 実行環境

- Windows 10
- Python 3.6.3
- luigi 2.7.2
