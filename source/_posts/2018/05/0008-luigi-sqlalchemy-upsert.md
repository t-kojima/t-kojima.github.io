---
title: '[Python] luigiとSQLAlchemyでUpsert'
date: 2018-05-15 21:40:05
tags:
- python
- luigi
- sqlalchemy
---

luigiのタスクの`output`をDB登録とする時、前回まででは`SQLAlchemy`との連携を使用した。
その中でタスクの`complete`判定は以下のようになっている。

<!-- more -->

- `complete`判定用に`table_updates`テーブルをDB上に持つ。このテーブルは最初に参照された時に自動生成される。
- 判定は`table_updates`テーブルに`update_id`をキーとしたレコードの有無で判断する。あれば実行済みだ。
- `update_id`はデフォルトでは`<タスクネームスペース>.<タスク名>_<パラメータ（の一部）>_<パラメータのmd5ハッシュ>`をキーとして使用する。
- `complete`が`False`の場合、データを対象テーブルにInsertする。

ただしこのデフォルト設定にはいくつか問題点がある。

&nbsp;

## **update_id**が固定

固定というわけではないが、タスクとパラメータが同一なら`complete`は`True`を返してしまい、`run`が走らずに依存元タスクへ処理が戻ってしまう。
これの対応は簡単で、`update_id`をオーバーライドして自分で定義してしまえばいい。前回の例ならば「ファイルパス」をキーとしてしまえばよい。

```python
    def update_id(self):
        return self.path
```

&nbsp;

## Insertが失敗する可能性がある

`update_id`は実際に登録するデータのキーと紐づいているわけではないので、`complete`が`False`になったとしてもデータ登録が`Duplicate Key Error`になる可能性がある。ここは`update_id`をプライマリーキーの値にする（もしくはユニークとなる値）よう自力で設定するしかない。

&nbsp;

## InsertではなくUpdateしたい

今回の本題がこれ
データをInsertして`Duplicate Key Error`になる時は、そもそもInsertではなくUpdateしたいという場合だ。（いわゆる`Upsert`）

MySQLの`ON DUPLICATE KEY UPDATE`機能をSQLAlchemyから呼び出して対応してみたい。

[sqla.CopyToTableのソースコードを確認](http://luigi.readthedocs.io/en/stable/_modules/luigi/contrib/sqla.html#CopyToTable.copy)すると、Insert処理をしているのは`copy`メソッドであることがわかる。このメソッドのコメントに以下のようなことが書いてある。

>   A task that needs row updates instead of insertions should overload this method.

Updateにしたければオーバーロードしろと  
オーバーライドじゃないのか…？と思いつつ実装してみる

&nbsp;

### オーバーライド後

前回のコードに適用するとこんな感じだ

```python
from sqlalchemy.dialects.mysql import insert

class Archive(sqla.CopyToTable):
    path = luigi.Parameter()
    reflect = True
    connection_string = 'mysql://<user>:<password>@<server>/<db_name>?charset=utf8'
    table = "archives"

    def requires(self):
        return BuildRecord(path)
    
    def update_id(self):
        return self.path

    def copy(self, conn, ins_rows, table_bound):
        bound_cols = dict((c, sqlalchemy.bindparam("_" + c.key)) for c in table_bound.columns)
        ins = insert(table_bound).values(bound_cols)
        on_duplicate_key = ins.on_duplicate_key_update(
            filename=ins.inserted.filename,
            filepath=ins.inserted.filepath,
            created_at=ins.inserted.created_at,
            updated_at=ins.inserted.updated_at
        )
        conn.execute(on_duplicate_key, ins_rows)
```

オーバーライド前はこのようなコードだったので、一行ずつ比較して見てみたい。

```python
# luigi/contrib/sqla.py
    def copy(self, conn, ins_rows, table_bound):
        bound_cols = dict((c, sqlalchemy.bindparam("_" + c.key)) for c in table_bound.columns)
        ins = table_bound.insert().values(bound_cols)
        conn.execute(ins, ins_rows)
```

&nbsp;

#### 1行目

```python
bound_cols = dict((c, sqlalchemy.bindparam("_" + c.key)) for c in table_bound.columns)
```

投入されるデータ（`ins_rows`）はカラム名に`_`がついているので、そのままではカラム名不一致でテーブルに投入できない。その為、カラム名の対応表として上記のdictを作成している。
この行は特にいじらず、このままにしておく。

&nbsp;

#### 2行目

```python
ins = table_bound.insert().values(bound_cols)
```

`values`にカラム名の対応表（1行目で作成したdict）を設定することで、パラメータを更新している。しかしこのままではMySQLに対応していないので`sqlalchemy.dialects.mysql`の`insert`を利用して再定義する。

```python
from sqlalchemy.dialects.mysql import insert

ins = insert(table_bound).values(bound_cols)
on_duplicate_key = ins.on_duplicate_key_update(
    filename=ins.inserted.filename,
    filepath=ins.inserted.filepath,
    created_at=ins.inserted.created_at,
    updated_at=ins.inserted.updated_at
)
```

すると`on_duplicate_key_update`メソッドが使用できるので、`Duplicate Key`の時にUpdateをするカラムを設定することができる。

&nbsp;

#### 3行目

```python
conn.execute(ins, ins_rows)
```

ここは単純にSQLを流している。以下のように書き換えてあげよう。

```python
conn.execute(on_duplicate_key, ins_rows)
```

以上で`Upsert`を実装することができた。
いままで`Upsert`って使ったことがなかったけど、これってluigiだけじゃなくて普通のSQLAlchemyでも使えるね


MySQL以外にも、PostgreSQLやSQLiteでもできるらしいが、今回はここまで


&nbsp;

## 参考

- [SQLAlchemy で MySQL の upsert を試す](https://qiita.com/elm200/items/5ba61d8799da99b8d162)  
- [INSERT…ON DUPLICATE KEY UPDATE (Upsert)](http://docs.sqlalchemy.org/en/latest/dialects/mysql.html#insert-on-duplicate-key-update-upsert)  
- [MySQL DML Constructs](http://docs.sqlalchemy.org/en/latest/dialects/mysql.html#sqlalchemy.dialects.mysql.dml.insert)

&nbsp;

## 実行環境

- Windows 10
- Python 3.6.3
- luigi 2.7.2
- SQLAlchemy 1.2.1


