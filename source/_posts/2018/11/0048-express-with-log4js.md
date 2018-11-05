---
title: '[Node] ExpressでAPIを作ったらログも吐く'
date: 2018-11-05 11:45:55
categories:
  - ['Node', 'Express'] 
tags:
  - nodejs
  - express
  - log4js
---

[[Node] Express を使って最速で JSON を返す](https://t-kojima.github.io/2018/11/02/0047-helloworld-express-with-json/)の続き

API が動いたらログを吐かなきゃ（使命感）

<!-- more -->

# サンプルリポジトリ

<a href="https://github.com/t-kojima/example-express-api" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

# ログを出力する

[log4js](https://github.com/log4js-node/log4js-node)というモジュールを使ってログを吐くようにする。

```bash
yarn add log4js
```

まずは最小の構成でログを吐いてみる。

```js
const express = require('express');
const app = express();
const log4js = require('log4js'); // 追加
const logger = log4js.getLogger(); // 追加

logger.level = 'info'; // 追加

app.get('/', (req, res) => res.send('hello world!'));

app.get('/:name', (req, res) => {
  res.header('Content-Type', 'application/json; charset=utf-8');
  res.send({ message: `hello ${req.params.name}` });
  logger.info(`hello ${req.params.name}`); // 追加
});

app.listen(3000, () => console.log('http://localhost:3000'));
```

これで`http://localhost:3000/fuga`にアクセスすると、以下のログがコンソールに表示される。

```bash
[2018-11-05T09:05:17.473] [INFO] default - hello fuga
[2018-11-05T09:05:17.519] [INFO] default - hello favicon.ico
```

`favicon.ico`…？リクエスト毎に`http://localhost:3000/favicon.ico`は必ずリクエストが走るから、かな？

## ファイルに出力

コンソールに吐くだけだと`console.log`と変わらないので、ファイルに吐くようにしよう。

`log4js.configure`で appender と category を設定し、`./app.log`に info ログを出力するようにする。

```js
const express = require('express');
const app = express();
const log4js = require('log4js');

log4js.configure({
  appenders: {
    app: {
      type: 'file',
      filename: './app.log',
    },
  },
  categories: {
    default: { appenders: ['app'], level: 'info' },
  },
});
const logger = log4js.getLogger();

app.get('/', (req, res) => res.send('hello world!'));

app.get('/:name', (req, res) => {
  res.header('Content-Type', 'application/json; charset=utf-8');
  res.send({ message: `hello ${req.params.name}` });
  logger.info(`hello ${req.params.name}`);
});

app.listen(3000, () => console.log('http://localhost:3000'));
```

`log4js.getLogger()`を呼んだとき、引数なしならば categories の default が呼ばれる。

また、categories で level も指定するようにしたので、`logger.level = 'info';`の行は不要になる。

これで実行し、アクセスすると。。。

```bash
$ node app
http://localhost:3000

$ curl http://localhost:3000/fuga
```

app.log ファイルに以下のログが出力される。

```log
[2018-11-05T10:30:34.829] [INFO] default - hello fuga
[2018-11-05T10:30:34.866] [INFO] default - hello favicon.ico
```

## Express と連携

イイ感じになってきたが、もうちょっと行ってみよう。Express が出力するログを log4js に連携させてみる。

完成形は以下の形

```js
const express = require('express');
const app = express();
const log4js = require('log4js');

log4js.configure('./log4js.json'); // 修正
const logger = log4js.getLogger();

app.use(log4js.connectLogger(log4js.getLogger('express'))); // 追加

app.get('/', (req, res) => res.send('hello world!'));

app.get('/:name', (req, res) => {
  res.header('Content-Type', 'application/json; charset=utf-8');
  res.send({ message: `hello ${req.params.name}` });
  logger.info(`hello ${req.params.name}`);
});

app.listen(3000, () => console.log('http://localhost:3000'));
```

まず`log4js.configure`の設定だが内容が多くなってくると見通しが悪くなるので、設定を json を読み込む形にする。

以下の内容で log4js.json を用意して

```json
{
  "appenders": {
    "console": { "type": "console" },
    "app": { "type": "file", "filename": "app.log" },
    "express": { "type": "file", "filename": "express.log" }
  },
  "categories": {
    "default": { "appenders": ["console", "app"], "level": "debug" },
    "express": { "appenders": ["console", "express"], "level": "info" }
  }
}
```

`log4js.configure('./log4js.json');`で読み込むようにする。そして express の出力を連携する為に、`app.use(log4js.connectLogger(log4js.getLogger('express')));`の一行を追加する。

ちなみに`"console": { "type": "console" }`という設定も追加し、default,express の両方の category の appenders に追加しているので、ファイル出力すると共にコンソールにも出力される。

これもまた実行すると。。。

```bash
$ node app
http://localhost:3000

$ curl http://localhost:3000
[2018-11-05T10:37:39.798] [INFO] express - ::1 - - "GET / HTTP/1.1" 304 - "" "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:62.0) Gecko/20100101 Firefox/62.0"
[2018-11-05T10:37:40.026] [INFO] default - hello favicon.ico
[2018-11-05T10:37:40.029] [INFO] express - ::1 - - "GET /favicon.ico HTTP/1.1" 200 31 "" "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:62.0) Gecko/20100101 Firefox/62.0"
```

このようにコンソールにも出力されて

```log
[2018-11-05T10:37:39.798] [INFO] express - ::1 - - "GET / HTTP/1.1" 304 - "" "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:62.0) Gecko/20100101 Firefox/62.0"
[2018-11-05T10:37:40.029] [INFO] express - ::1 - - "GET /favicon.ico HTTP/1.1" 200 31 "" "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:62.0) Gecko/20100101 Firefox/62.0"
```

express.log としてログファイルにも出力される！

# Tips

## 日付でログをローテーション

実運用する場合、日付でログをローテーションする運用が非常に多いので、設定のやり方を残しておく。

```json
{
  "appenders": {
    "app": {
      "type": "dateFile",
      "filename": "app.log"
    }
  },
  "categories": {
    "default": { "appenders": ["app"], "level": "debug" }
  }
}
```

各パラメータの意味は以下の通り

| パラメータ           | 必須 | デフォルト値 | 備考                                                                                                     |
| -------------------- | ---- | ------------ | -------------------------------------------------------------------------------------------------------- |
| type                 | 〇   | -            | **dateFile**にする。しないと日付にならない。                                                             |
| filename             | 〇   | -            | ベースとなるファイル名                                                                                   |
| pattern              |      | .yyyy-MM-dd  | ファイル名に付く日付フォーマット、省略するとデフォルト値になる。                                         |
| encoding             |      | utf-8        | ファイルエンコーディング                                                                                 |
| mode                 |      | 0644         | ファイルモード                                                                                           |
| flags                |      | a            | 恐らくファイルオープンのモード、"a"は"append"と思われる。。。                                            |
| compress             |      | false        | ローテーションのタイミングで、`.gz`で圧縮する。                                                          |
| alwaysIncludePattern |      | false        | ローテーション前のログファイルにも日付フォーマットを適用する。                                           |
| daysToKeep           |      | 0            | 0 以上の場合、ローテーションのタイミングで（値）日以前のログファイルは削除する。                         |
| keepFileExt          |      | false        | ローテーションした時にファイル拡張子を保持する。`app.log.2018-11-02`ではなく`app.2018-11-02.log`になる。 |

**alwaysIncludePattern**の補足：false の場合ログは`app.log`（カレント）に書き込まれていき、ローテーションのタイミングで`app.log.yyyy-MM-dd`にバックアップされる。そしてカレントのログは`app.log`のままリフレッシュされ、当日分のログはまたここに書き込まれていく。true にすると最初から`app.log.yyyy-MM-dd`（カレント）としてログが作成され、ローテーションされると翌日の日付のファイルがカレントになり、書き込まれていく。

# 実行環境

- Node 8.11.3
- Express 4.16.4
- log4js 3.0.6

# 参考

- [express というか node というか・・でログを取る（log4js 編） - Qiita]h(ttps://qiita.com/zaburo/items/eaac8df099455f7367f7)
- [log4js-node by log4js-node](https://log4js-node.github.io/log4js-node/)
- [Date Rolling File Appender - log4js-node](https://log4js-node.github.io/log4js-node/dateFile)
