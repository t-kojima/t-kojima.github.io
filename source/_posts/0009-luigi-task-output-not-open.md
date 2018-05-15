---
title: '[Python] luigiのTask#outputをopenで書き込まない場合'
date: 2018-05-15 21:44:18
tags:
- python
- luigi
---

ちょっとニッチな状況でハマったのでメモ  

`run`でファイルコピーするだけのタスクを作った時、ファイルコピーを以下のようにやってしまった。

<!-- more -->

```python

import shutil
from luigi import Task

class UserTask(Task):

    def run(self):
        shutil.copy('<source path>', self.output().path)

    def output(self):
        return LocalTarget('<output path>')

```

<br/>

## ディレクトリを自動で作ってくれない

`shutil.copy`でコピーしているので当然と言えば当然だが、`output`の親ディレクトリを自動で作成はしてくれない。
`self.output().makedirs()`を明示的に実行する必要がある。

```python
    def run(self):
        self.output().makedirs()
        shutil.copy('<source path>', self.output().path)
```

`self.output().makedirs()`は`self.output().open('w')`で書き込む時は内部的に自動で呼んでくれる。なので`shutil.copy`ではなく直接書き込めば`makedirs()`を明示的に呼ぶ必要はない。

```python
    def run(self):
        with self.output.oepn('w') as output, open('<source path>', 'r') as input:
            output.write(input.read())
```

## ユニットテスト時にMockTargetでパッチできない

パッチできないわけではないけど、意図した動きにならなかった。

```python
# !/usr/bin/env python3
# coding: utf-8

import unittest
from unittest import mock
from luigi import Task
from luigi import build, LocalTarget
from luigi.mock import MockTarget

class UserTask(Task):

    def run(self):
        self.output().makedirs()
        shutil.copy('<source path>', self.output().path)

    def output(self):
        return LocalTarget('<output path>')


def mocked_output(*args, **kwargs):
    m = MockTarget('mock_target')
    m.makedirs = lambda: True
    return m


class TestUserTask(unittest.TestCase):

    @mock.patch('__main__.UserTask.output', side_effect=mocked_output)
    def test_method(self, m_mock):
        """ test file copy """
        result = build([UserTask()], local_scheduler=True)
        self.assertTrue(result)


if __name__ == "__main__":
    unittest.main()
```

`LocalTarget`を`MockTarget`に置き換えるよう`output`をモックでパッチしたかった。

しかし`output`ファイルの書き出しは`self.output().open`でやっているわけではないので、`MockTarget`にしたにも関わらずファイルの実体が書き出されてしまう。

ここも`shutil.copy`ではなく`self.output().open('w')`でやれば問題なかった。

## パッチはするけどLocalTargetで

モックでパッチするけど、`MockTarget`ではなく`LocalTarget`で`is_tmp`する形にしている。

```python
def mocked_output(*args, **kwargs):
    m = LocalTarget(is_tmp=True)
    m.makedirs = lambda: True
    return m
```

また、`makedirs`が呼ばれるとディレクトリを作ったしまうので、適当なメソッド（上記ではラムダ）で上書きしておく

## まとめ

- `output`のターゲットは、ちゃんと`self.output().open('w')`で書かないと意図しない動作することがあるよ！

## 実行環境

- Windows 10
- Python 3.6.3
- luigi 2.7.2
