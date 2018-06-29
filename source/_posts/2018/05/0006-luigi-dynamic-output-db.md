---
title: '[Python] luigiで動的アウトプット（DB登録）'
date: 2018-05-15 20:57:14
tags:
- python
- luigi
- sqlalchemy
---

前回の残課題

- `Archive`はDB登録する処理だから、`output`は[MySqlTarget](http://luigi.readthedocs.io/en/stable/api/luigi.contrib.mysqldb.html)にしたほうがいい

どうやって`output`をDB登録にするか、普段`SQLAlchemy`を使っているので`SQLAlchemy`との連携をやりたいと思う。

<!-- more -->

## 前回のコード

前回のコードは以下のものだ、`output`がファイル出力になっているのでこれをDB登録に対応させていく

```python
import luigi

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

参考にしたのは
[Python: データパイプライン構築用フレームワーク Luigi を使ってみる](http://blog.amedama.jp/entry/2017/05/13/203907)と[Using SQLAlchemy in Luigi Workflow Pipeline](http://gouthamanbalaraman.com/blog/sqlalchemy-luigi-workflow-pipeline.html)と[公式ドキュメント](http://luigi.readthedocs.io/en/stable/api/luigi.contrib.sqla.html)

## データベースの準備

[SQLAlchemy Migrate](https://sqlalchemy-migrate.readthedocs.io/en/latest/)を使って`MySQL`のデータベースを準備する。
テーブル定義はこんな感じ

```python
from sqlalchemy import *
from migrate import *
from sqlalchemy.dialects.mysql import BIGINT, DATETIME, TEXT

meta = MetaData()
table = Table(
    'archives', meta,
    Column('id', BIGINT(unsigned=True), primary_key=True),
    Column('filename', TEXT, nullable=False, unique=True),
    Column('filepath', TEXT, nullable=False),
    Column('created_at', DATETIME, nullable=False),
    Column('updated_at', DATETIME, nullable=False),
    mysql_charset='utf8')


def upgrade(migrate_engine):
    meta.bind = migrate_engine
    table.create()


def downgrade(migrate_engine):
    meta.bind = migrate_engine
    table.drop()
```



## Dynamic dependencies

ベーシックなサンプルは以下の通りだ（テーブルは既にあるので`reflect = True`のパターン）

```python
class SQLATask(sqla.CopyToTable):
    reflect = True
    connection_string = "sqlite://"
    table = "item_property"

    def rows(self):
        for row in [("item1" "property1"), ("item2", "property2")]:
            yield row
```

しかしこのサンプルをそのまま適用することはできない…動的依存関係の解決の為に`run()`を定義する必要があるからだ。

なので、ここでは**データを登録するタスク**と**データを準備するタスク**を分けるサンプルが参考になる。
**データを準備するタスク**として`BuildRecord`タスクを定義し、動的依存関係の解決はそのタスクの`run()`で行うのだ。

```python
import luigi
from luigi.contrib import sqla
from luigi.mock import MockTarget
from pathlib import Path
from datetime import datetime

class BuildRecord(luigi.Task):

    def run(self):
        pickup = Pickup()
        yield pickup
        with pickup.output().open('r') as f:
            input_path = f.read()

        filecopy = FileCopy(input_path)
        yield filecopy
        output_path = Path(filecopy.output().path)

        with self.output().open('w') as f:
            f.write('0\t{0}\t{1}\t{2}\t{3}'.format(
                output_path.name, output_path.as_posix(), datetime.now(), datetime.now()))

    def output(self):
        return MockTarget("BuildRecord")


class Archive(sqla.CopyToTable):

    reflect = True
    connection_string = 'mysql://<user>:<password>@<server>/<db_name>?charset=utf8'
    table = "archives"

    def requires(self):
        return BuildRecord()
```

これでDB登録できるようなった！

基本的な流れは今まで通り、`Archive`タスクを実行すると`requires`で`BuildRecord`を呼び、`BuildRecord`の`run`でデータを作成、`Archive`に戻ってSQLAlchemyでDB登録という形だ。

細かな注意点は以下の通りだ

- 登録するデータは`\t`で区切った文字列で渡す。今回は1行のみだが、複数行渡す場合は`\n`で改行する。
- 区切り文字`\t`は`column_separator`オプションで変更可能
- `id`は*AUTO INCREMENT*だが、登録データを空白にするとカラムがずれるので`0`を指定してあげる。そうするとInsertする時にちゃんと*AUTO INCREMENT*してくれる。

## `sqla.CopyToTable`の`output`

`Archive`タスクは`output`としてレコードをDBに登録することができたが、このレコードを削除したら次にタスクを回した時再度登録されるだろうか？

実際やってみると再登録はされない。それどころか`Archive`処理自体が走らなくなる。

`sqla.CopyToTable`の`complete`をどうやって判定しているかという話になるが、`update_id`というメソッドで判定しており、内部で`task_id`を参照している。`task_id`はタスクを走らせた時のログに出てくる`Archive__fbe666d9fe`のような値で、`<タスク>_<パラメータ>_<パラメータのmd5ハッシュ>`（正確じゃないかもしれないけどこんな感じ）というフォーマットになっている。

つまりタスクとパラメータが同一ならキーが変わらないので`sqla.CopyToTable`は走らない。
そしてこの情報は`table_updates`というテーブルが自動的に作成されて保存されている。

今回の要件だとタスクのIDよりもレコードの内容で`complete`を判定して欲しい。具体的にはユニークである`filename`のカラムだ。
しかし`Archive`タスクで`update_id`メソッドをオーバーライドしようとしても、`complete`を判定するタイミングで`filename`はわからない。ジレンマの再来だ。



## 残課題

- `update_id`をオーバーライドして、レコード内容をもとに`complete`判定をするようにしたい。現状ではパラメータを渡さなければ一回限りの実行しかしないし、何かしらパラメータを渡しても無条件で`run`が走る形になってしまう。



## 余談

サンプルでも`MockTarget`使ってる…`MockTarget`はユニットテスト用途と言われているけど、やっぱり`output`がファイルである必要性が無いタスクは`MockTarget`使ったほうがスマートだよね


## 実行環境

- Windows 10
- Python 3.6.3
- luigi 2.7.2
- SQLAlchemy 1.2.1
- sqlalchemy-migrate 0.11.0
