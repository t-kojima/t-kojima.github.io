---
title: '[Python] luigiで動的アウトプットはしない'
date: 2018-05-15 20:57:30
tags:
- python
- luigi
---

前回までの記事のちゃぶ台返しです。

そもそも「依存先タスクの`output`を元に、依存元の`output`パスが決定する」という要件の場合、本来`output`が存在するかどうかを判断して、`requires`（および`run`）を実行するというフローのところ、「`requires`を解決しないと`output`がわからない」という逆転現象が起きてしまう。

<!-- more -->

前回までで luigi の動的アウトプット（Dynamic dependencies）でなんとか凌ごうとしていたが、依存関係が増える毎に上記の問題が頻発しうることが分かった。

ファイルのピックアップ処理が依存関係を辿った時に一番上流にあるからといって、無理やり luigi のタスクに落とし込んだところが失敗だったのかなと思う。

## 修正後のコード

修正後はファイルのピックアップ処理をタスクのスタートよりも前に持ってきていて、タスク開始時のパラメータとしてパスを渡してしまう形にした。

```bash
#.tree
.
├── main.py
└── tasks
    ├── archive.py
    └── file_copy.py
```

```python
# main.py

import luigi
from tasks.archive import Archive


class Main(luigi.WrapperTask):

    def pickup(self, params):
        # ファイルピックアップ処理
        return '<ファイルパス>'

    def requires(self):
        path = self.pickup()
        return Archive(path)


def main():
    luigi.build([Main()], local_scheduler=True)


if __name__ == '__main__':
    main()
```

`Main`タスク、一連の処理のキーとなるファイルパスを取得し、`Archive`タスクへパラメータとして渡しつつキックする。

```python
# tasks/archive.py

import luigi
from luigi.contrib import sqla
from luigi.mock import MockTarget
from pathlib import Path
from datetime import datetime
from tasks.file_copy import FileCopy


class BuildRecord(luigi.Task):
    path = luigi.Parameter()

    def requires(self):
        return FileCopy(self.path)

    def run(self):
        output_path = Path(self.input().path)

        with self.output().open('w') as f:
            f.write('0\t{0}\t{1}\t{2}\t{3}'.format(
                output_path.name, output_path.as_posix(), datetime.now(), datetime.now()))

    def output(self):
        return MockTarget("BuildRecord")


class Archive(sqla.CopyToTable):
    path = luigi.Parameter()
    reflect = True
    connection_string = 'mysql://<user>:<password>@<server>/<db_name>?charset=utf8'
    table = "archives"

    def requires(self):
        return BuildRecord(path)

    def update_id(self):
        return self.path
```

`Archive`タスク、`update_id`メソッドをオーバーライドしてファイルパスをキーにして`complete`判定を下すようにする。また、`FileCopy`タスクも動的ではなく通常の依存関係処理で呼ぶようにする。

`BuildRecord`タスクは元々動的依存関係を解決する為に用意したタスクだが、登録データを整形する為残している。`Archive`タスクと統合しても良いが、その場合は`rows`メソッドをオーバーライドして、その中でデータの整形をすべきだろう。

```python
# tasks/file_copy.py

import luigi
import shutil


class FileCopy(luigi.Task):
    path = luigi.Parameter()

    def run(self):
        shutil.copy2(self.path, self.output().path)

    def output(self):
        # self.pathを元にアウトプットパスを作る
        # 例えば以下のような形
        # output_path = '/path/to/dst' / Path(self.path).name
        return luigi.LocalTarget('<アウトプットパス>')
```

`FileCopy`タスク、ファイルパスをパラメータで受け取りファイルコピーを行う。`Pickup`タスクが無くなったので、このタスクが最上流になる。

---

前回のコードよりもスッキリしたと思う。

## 実行環境

- Windows 10
- Python 3.6.3
- luigi 2.7.2
- SQLAlchemy 1.2.1
- sqlalchemy-migrate 0.11.0
