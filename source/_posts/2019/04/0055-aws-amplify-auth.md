---
title: '[AWS Amplify] ポーカーつくろうぜ！（プロジェクト初期設定）'
date: 2019-04-30 09:23:10
categories:
  - AWS
tags:
  - aws-amplify
---

GW の 10 連休で時間があるので AWS Amplify をやてみる。

<!-- more -->

## 題材

特にアイディアがない問題なので、またカードゲームでも作ってみよう。
以前はブラックジャックだったので、今度はポーカーあたりでどうだろうか？

## aws-amplify CLI - アカウント設定

アカウントとの紐付けなどの初期セットアップは以下の手順が非常に詳しくそのまま使えるので、これを参考にやる。

参考：[AWS Amplify CLI の使い方〜インストールから初期セットアップまで〜 - Qiita](https://qiita.com/Junpei_Takagi/items/f2bc567761880471fd54)

以下コンソールダイジェスト

```bash
$ yarn global add @aws-amplify/cli
$ amplify configure
Follow these steps to set up access to your AWS account:

Sign in to your AWS administrator account:
https://console.aws.amazon.com/
Press Enter to continue

Specify the AWS Region
? region:  us-east-1
Specify the username of the new IAM user:
? user name:  amplify-SCCTT
Complete the user creation using the AWS console
https://console.aws.amazon.com/iam/home?region=undefined#/users$new?step=final&accessKey&userNames=amplify-SCCTT&permissionType=policies&policies=arn:a
ws:iam::aws:policy%2FAdministratorAccess
Press Enter to continue

Enter the access key of the newly created user:
? accessKeyId:  ******************************
? secretAccessKey:  ****************************************
This would update/create the AWS Profile in your local machine
? Profile Name:  practice-poker

Successfully set up the new user.
```

### Amplify の初期設定

`amplify init`で Amplify の初期設定をする。

```bash
$ amplify init
```

対話式で色々質問されるので回答してく

```
? Enter a name for the project
> practice-poker

? Enter a name for the environment
# environmentってなんだろう？とりあえずdevelopとかにしとく
> develop

? Choose your default editor:
> Visual Studio Code

? Choose the type of app that you're building
> javascript

Please tell us about your project
? What javascript framework are you using
> react

? Source Directory Path:
> src

? Distribution Directory Path:
> build

? Build Command:
# これでいいのか？自信なし。。。
> yarn build

? Start Command:
> yarn start

? Do you want to use an AWS profile?
> Yes
? Please choose the profile you want to use
> practice-poker
```

全ログは以下の通り

```
$ amplify init

Note: It is recommended to run this command from the root of your app directory
? Enter a name for the project practice-poker
? Enter a name for the environment develop
? Choose your default editor: Visual Studio Code
? Choose the type of app that you're building javascript
Please tell us about your project
? What javascript framework are you using react
? Source Directory Path:  src
? Distribution Directory Path: build
? Build Command:  yarn build
? Start Command: yarn start
Using default provider  awscloudformation

For more information on AWS Profiles, see:
https://docs.aws.amazon.com/cli/latest/userguide/cli-multiple-profiles.html

? Do you want to use an AWS profile? Yes
? Please choose the profile you want to use practice-poker
⠙ Initializing project in the cloud...

CREATE_IN_PROGRESS practice-poker-20190430105032 AWS::CloudFormation::Stack Tue Apr 30 2019 10:50:34 GMT+0900 (JST) User Initiated
CREATE_IN_PROGRESS DeploymentBucket              AWS::S3::Bucket            Tue Apr 30 2019 10:50:39 GMT+0900 (JST)
CREATE_IN_PROGRESS AuthRole                      AWS::IAM::Role             Tue Apr 30 2019 10:50:39 GMT+0900 (JST)
CREATE_IN_PROGRESS UnauthRole                    AWS::IAM::Role             Tue Apr 30 2019 10:50:39 GMT+0900 (JST)
⠹ Initializing project in the cloud...

CREATE_IN_PROGRESS AuthRole         AWS::IAM::Role  Tue Apr 30 2019 10:50:40 GMT+0900 (JST) Resource creation Initiated
CREATE_IN_PROGRESS DeploymentBucket AWS::S3::Bucket Tue Apr 30 2019 10:50:40 GMT+0900 (JST) Resource creation Initiated
CREATE_IN_PROGRESS UnauthRole       AWS::IAM::Role  Tue Apr 30 2019 10:50:40 GMT+0900 (JST) Resource creation Initiated
⠙ Initializing project in the cloud...

CREATE_COMPLETE AuthRole   AWS::IAM::Role Tue Apr 30 2019 10:50:51 GMT+0900 (JST)
CREATE_COMPLETE UnauthRole AWS::IAM::Role Tue Apr 30 2019 10:50:52 GMT+0900 (JST)
⠹ Initializing project in the cloud...

CREATE_COMPLETE DeploymentBucket              AWS::S3::Bucket            Tue Apr 30 2019 10:51:01 GMT+0900 (JST)
CREATE_COMPLETE practice-poker-20190430105032 AWS::CloudFormation::Stack Tue Apr 30 2019 10:51:05 GMT+0900 (JST)
✔ Successfully created initial AWS cloud resources for deployments.
✔ Initialized provider successfully.
Initialized your environment successfully.

Your project has been successfully initialized and connected to the cloud!

Some next steps:
"amplify status" will show you what you've added already and if it's locally configured or deployed
"amplify <category> add" will allow you to add features like user login or a backend API
"amplify push" will build all your local backend resources and provision it in the cloud
"amplify publish" will build all your local backend and frontend resources (if you have hosting category added) and provision it in the cloud

Pro tip:
Try "amplify add api" to create a backend API and then "amplify publish" to deploy everything
```

ここまで完了すると、AWS の CloudFormation にスタックが作成されている。

![CloudFormationに自動作成されたスタック](/images/55-01.png)

## create-react-app をホスティング

やはり何かしらホスティングして、「できた！」ってところを確認したい。
なので create-react-app のデフォルトアプリをホスティングする。

しかしカレントディレクトリにアプリを作成しようとするとコンクリフトする。

```
$ create-react-app .

The directory . contains files that could conflict:

  __src
  amplify
  build
  node_modules
  package.json
  public
  src
  yarn.lock

Either try using a new directory name, or remove the files listed above.
```

そういえば create-react-app はそうだった。
しょうがないのでサブディレクトリに一旦展開してカレントに持ってくる。

```
$ create-react-app temp
# 生成されたファイルをルートに移動
```

create-react-app 使う場合、git clone 直後にやっといたほうがよかったなとちょっと後悔。

### amplify-js

aws-amplify のプロジェクトを作るには、amplify-js を導入する必要がある。
firebase でいう firebase-tools みたいなもの（？）だ。たぶん

<a href="https://github.com/aws-amplify/amplify-js" class="embedly-card" data-card-image="0" data-card-controls="0" data-card-align="left"></a>

yarn で入れる。React も使うので`aws-amplify-react`も入れる。

```bash
yarn add aws-amplify
yarn add aws-amplify-react
```

### ホスティング有効化

ホスティングを有効化する。

```bash
amplify add hosting
```

こんな感じで使いたいサービスを追加していくらしい。

### デプロイ

`amplicy publish`でデプロイ！続けるかどうかだけ聞かれるので YesYes

```bash
$ amplify publish

Current Environment: develop

| Category | Resource name   | Operation | Provider plugin   |
| -------- | --------------- | --------- | ----------------- |
| Hosting  | S3AndCloudFront | Create    | awscloudformation |
? Are you sure you want to continue? Yes
⠸ Updating resources in the cloud. This may take a few minutes...

UPDATE_IN_PROGRESS practice-poker-20190430110238 AWS::CloudFormation::Stack Tue Apr 30 2019 11:33:09 GMT+0900 (JST) User Initiated
⠸ Updating resources in the cloud. This may take a few minutes...

CREATE_IN_PROGRESS hostingS3AndCloudFront AWS::CloudFormation::Stack Tue Apr 30 2019 11:33:16 GMT+0900 (JST)
CREATE_IN_PROGRESS hostingS3AndCloudFront AWS::CloudFormation::Stack Tue Apr 30 2019 11:33:18 GMT+0900 (JST) Resource creation Initiated
⠼ Updating resources in the cloud. This may take a few minutes...

CREATE_IN_PROGRESS practice-poker-20190430110238-hostingS3AndCloudFront-105QEPY9AH53T AWS::CloudFormation::Stack Tue Apr 30 2019 11:33:17 GMT+0900 (JST) User Initiated
⠴ Updating resources in the cloud. This may take a few minutes...

CREATE_IN_PROGRESS S3Bucket AWS::S3::Bucket Tue Apr 30 2019 11:33:23 GMT+0900 (JST)
⠦ Updating resources in the cloud. This may take a few minutes...

CREATE_IN_PROGRESS S3Bucket AWS::S3::Bucket Tue Apr 30 2019 11:33:26 GMT+0900 (JST) Resource creation Initiated
⠇ Updating resources in the cloud. This may take a few minutes...

CREATE_COMPLETE S3Bucket AWS::S3::Bucket Tue Apr 30 2019 11:33:47 GMT+0900 (JST)
⠦ Updating resources in the cloud. This may take a few minutes...

CREATE_COMPLETE hostingS3AndCloudFront AWS::CloudFormation::Stack Tue Apr 30 2019 11:33:52 GMT+0900 (JST)
⠦ Updating resources in the cloud. This may take a few minutes...

UPDATE_COMPLETE_CLEANUP_IN_PROGRESS practice-poker-20190430110238 AWS::CloudFormation::Stack Tue Apr 30 2019 11:33:56 GMT+0900 (JST)
UPDATE_COMPLETE                     practice-poker-20190430110238 AWS::CloudFormation::Stack Tue Apr 30 2019 11:33:57 GMT+0900 (JST)
✔ All resources are updated in the cloud

Hosting endpoint: http://practice-poker-20190430112522-hostingbucket-develop.s3-website-ap-northeast-1.amazonaws.com

yarn run v1.12.3
$ react-scripts build
Creating an optimized production build...
Compiled successfully.

File sizes after gzip:

  36.44 KB  build/static/js/2.b41502e9.chunk.js
  762 B     build/static/js/runtime~main.a8a9905a.js
  602 B     build/static/js/main.28647029.chunk.js
  523 B     build/static/css/main.584f321a.chunk.css

The project was built assuming it is hosted at the server root.
You can control this with the homepage field in your package.json.
For example, add this to build it for GitHub Pages:

  "homepage" : "http://myname.github.io/myapp",

The build folder is ready to be deployed.
You may serve it with a static server:

  yarn global add serve
  serve -s build

Find out more about deployment here:

  https://bit.ly/CRA-deploy

✨  Done in 5.26s.
frontend build command exited with code 0
✔ Uploaded files successfully.
Your app is published successfully.
http://practice-poker-20190430112522-hostingbucket-develop.s3-website-ap-northeast-1.amazonaws.com
```

いった！

![いつもの](/images/55-02.png)

http://practice-poker-20190430112522-hostingbucket-develop.s3-website-ap-northeast-1.amazonaws.com/

## 参考

- [AWS Amplify CLI の使い方〜インストールから初期セットアップまで〜 - Qiita](https://qiita.com/Junpei_Takagi/items/f2bc567761880471fd54)
- [AmplifyとReactを使用して爆速でWEBアプリを作成する - Qiita](https://qiita.com/oliverSI_/items/efc1a1e90584baac9e61)
