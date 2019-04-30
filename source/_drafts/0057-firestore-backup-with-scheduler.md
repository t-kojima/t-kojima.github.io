---
title: 0057-firestore-backup-with-scheduler
date: 2019-03-02 15:27:13
categories:
  - Firebase
tags:
  - firebase
  - firestore
---

<!-- more -->

https://medium.com/google-cloud-jp/firesto-77272bac8762
https://qiita.com/Kasanomo/items/566191f0c106b4669f22


https://developers-jp.googleblog.com/2017/04/how-to-schedule-cron-jobs-with-cloud.html

ここの図がわかりやすい

```
AppEngine -> Cloud Pub/Sub -> Cloud Functions
```

https://qiita.com/miya0001/items/ad0412ca379505088339

上記の参考サイトではGAEのアプリを丁寧に説明してくれているが、サンプルリポジトリがそのまま使えそうなので、以下をクローンしてベースにする。

https://github.com/FirebaseExtended/functions-cron

