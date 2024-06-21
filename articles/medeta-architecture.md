---
title: "めでた！の技術構成"
emoji: "🌱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "dart", "go", "openapi", "googlecloud"]
published: false
---

## はじめに

こんにちは！Altive株式会社のリードエンジニアの小林遼太（[@naipaka](https://twitter.com/naipakapaka)）です🦙

リリースしてから少し時間が経ってしまいましたが、4月に弊社でリリースした「めでた！」という家庭菜園をサポートするアプリの技術構成についてまとめました。

https://altive.co.jp/medeta

## アプリ概要

「めでた！」は、気軽に家庭菜園を始めて、楽しく続けられることを目指したアプリです。

現在のアプリの機能は以下の通りです。

- 家庭菜園のようすを撮影し、手軽に日記を書き残す
- 他ユーザーの日記をみて、他の人の育て方を学んだり、成長の喜びを共有する
- 複数枚の画像からタイムラプス動画を作成する

弊社ではFlutterでのアプリ開発をメインに行っていますが、よりスピーディーな開発を目指してサーバーサイドも自前で実装していく方針になりました。
今回のアプリがそれを実践した初めてのプロジェクトとなります！

詳しい背景は以下の記事をご覧ください。

https://zenn.dev/altiveinc/articles/generate-code-from-openapi


## 全体の構成

めでた！では、アプリやサーバーのコードを1つのリポジトリで管理する monorepo を採用しています。

```
medeta/
  ├─ firebase/ -- Firebase Hosting や Emulator Suite の設定ファイル
  ├─ medeta_app/ -- Flutter アプリのコード
  ├─ medeta_app_packages/ -- Flutter アプリで使う独自パッケージ
  ├─ medeta_app_console/ -- コンソールアプリのコード
  ├─ medeta_server/ -- Go サーバーのコード
  └─ openapi/ -- OpenAPI で定義した API のスキーマファイル
```

OpenAPI で API のスキーマを定義し、そのスキーマファイルを元にアプリ側の API クライアントとサーバー側の API インターフェースを生成しています。

monorepo で管理することで、サービス全体の見通しが良くなり、1人で1つの機能をアプリとサーバー横断して開発しやすくなりました。
実際に初期リリースするまではほぼ1人で開発を進めていましたが、アプリとサーバーのコードの行き来がしやすく、開発効率が向上したと感じています。

## アプリ

アプリは Flutter/Dart で開発しています。

プロジェクトの構成としては、Feature-first + Pages という形になっており、各機能ごとにディレクトリを分けつつ、UIに関するものは `pages` や `widgets` ディレクトリに格納しています。

通常のFeature-firstのアプローチでは、各ページもそれぞれの機能に含めることが多いと思いますが、1つのページが複数の機能にアクセスするケースがよくあったのでこのような形にしています。

また、本アプリで使用する独自パッケージを `medeta_app_packages` に格納しています。

```
medeta_app/
  └─lib/
    ├─ features/
    │   └─ diary/
    │       ├─ command/
    │       ├─ queries/
    │       └─ diary.dart
    ├─ pages/
    ├─ widgets/
    └─ main.dart
medeta_app_packages/
    ├─api_client/
    ├─app_lints/
    └─timelapse_creator/
```

### Navigation

アプリ内の画面遷移には [go_router](https://pub.dev/packages/go_router) と [go_router_builder](https://pub.dev/packages/go_router_builder) を使っています。
別プロジェクトで使用している [auto_route](https://pub.dev/packages/auto_route) も検討しましたが、導入してみて大きな問題には遭遇しなかったため、一旦 [go_router](https://pub.dev/packages/go_router) を採用した形です。

今後看過できない問題が発生した場合は、 [auto_route](https://pub.dev/packages/auto_route) に移行することも検討しています。

### 状態管理

https://docs.flutter.dev/data-and-backend/state-mgmt/ephemeral-vs-app

ephemeral state と app state を踏まえて、 [hooks_riverpod](https://pub.dev/packages/hooks_riverpod) と [flutter_hooks](https://pub.dev/packages/flutter_hooks) を使い分けて状態管理をしています。
また、Provider は [riverpod_generator](https://pub.dev/packages/riverpod_generator) を使って生成しています。

### API Client

API クライアントは OpenAPI で定義したスキーマファイルを元に [swagger_parser](https://pub.dev/packages/swagger_parser) を使って生成し、 `medeta_app_packages/api_client` に格納しています。

[swagger_parser](https://pub.dev/packages/swagger_parser) を使い始めた当初は Multipart リクエストに幾つかの問題がありましたが、最新のバージョンでは問題が解消されているため、問題なく使えるようになりました。

### 独自Linter

独自の Linter を作成し、 `medeta_app_packages/app_lints` に格納しています。

めでた！では多言語対応をしているのですが、その際に翻訳忘れが発生しないように、日本語の文字列を直書きしている場合に警告を出すようなルールを追加しています。

https://x.com/naipakapaka/status/1739651665505202657

まだルールは上記の1つしか追加していませんが、今後も開発効率が上がるようなルールを追加していく予定です。

### タイムラプス動画作成

メイン機能となるタイムラプス作成は、各ネイティブの API を使って実装し、Flutter 側から呼び出しています。

実装方法について別の記事でまとめているので、興味がある方は以下をご覧ください。

https://zenn.dev/altiveinc/articles/flutter-create-plugin-package-using-pigeon

## サーバー

サーバー側は Go で開発しています。

プロジェクトの構成は、様々な方のリポジトリを参考にしつつ手探りで進めているのですが、今のところ以下のような形になっています。

```
medeta_server/
  ├─ api/
  ├─ auth/
  ├─ config/
  ├─ db/
  ├─ entity/
  ├─ migrations/
  ├─ util/
  ├─ compose.yaml
  ├─ Dockerfile
  └─ main.go
```

### Web framework

https://echo.labstack.com/

APIを作るためのフレームワークには、 [Echo](https://echo.labstack.com/) を使っています。
他にも候補は多くありますが、初心者でも簡単に扱えて軽量かつ高性能であるという理由で採用しました。

### ORM/SQL Client

https://bun.uptrace.dev/

SQLを使ったデータベース操作には、[bun](https://bun.uptrace.dev/) を使用しています。
こちらも候補が多かったのですが、高速かつ SQL-first であるという理由で採用しました。
マイグレーションなどもサポートされており、不慣れな人でも扱いやすいと感じています。

### OpenAPI

https://github.com/oapi-codegen/oapi-codegen

OpenAPI で API のスキーマを定義し、そのスキーマファイルを元に [oapi-codegen](https://github.com/oapi-codegen/oapi-codegen) を使って API インターフェースを生成しています。

## インフラ

インフラは Google Cloud Platform / Firebase を使って構築しており、主に以下のサービスを利用しています。

- DB: Cloud SQL for PostgreSQL
- 認証: Firebase Authentication
- Storage: Firebase Storage

API のデプロイとローカルでの開発には、 [Cloud Code for VS Code](https://cloud.google.com/code/docs/vscode?hl=ja) を使っています。

https://cloud.google.com/code/docs/vscode?hl=ja

最新の Docker 環境で動かなくなったりと少し辛みを感じることもあるので、今後は [Cloud Code for VS Code](https://cloud.google.com/code/docs/vscode?hl=ja) に依存しないような開発環境を構築していく予定です。

また、ローカルで開発する際には [Firebase Emulator Suite](https://firebase.google.com/docs/emulator-suite?hl=ja) を使って開発しているため、その設定ファイルやデータも `firebase` ディレクトリに格納しています。

https://firebase.google.com/docs/emulator-suite?hl=ja

`task.json` を用いて VS Code でプロジェクトを開いたときに、自動で Docker コンテナで [Firebase Emulator Suite](https://firebase.google.com/docs/emulator-suite?hl=ja) と PostgreSQL が立ち上がるようにしているので、開発環境の構築がとても楽になっています。

## コンソールアプリ

アカウントBANや不適切な日記の非公開化など、管理者が行う操作を行うためのコンソールアプリも作成しています。

コンソールアプリも Flutter/Dart で作成しており、MacOSアプリとして配布できるようにしています。
一般的には管理画面を Web アプリとして作成することが多いかと思いますが、社内のみで使うものかつ、社員全員が Mac を使っているため、今回は試しに MacOS アプリとして作成してみることにしました。

作成した Mac アプリは `.dmg` ファイルをビルドして配布する形にし、万が一流出してしまった場合でも第三者が操作を行うことはできないように対応しています。

コンソールアプリはユーザーに直接即座に影響を与えるものではないため、改善や新しい機能の実装に比較的自由があります。
その結果、ジュニアエンジニアにとっては、Flutter や Go の開発スキルを身につける良い場となっています。
コンソールアプリの改善作業を通じて、実際のプロダクト改善に取り組む前に必要な経験を積むことができ、これがエンジニアのスキルアップに繋がると感じています。

## おわりに

めでた！を開発するにあたって初めてのことが多く、試行錯誤の連続でしたが、その分自分としても会社としても多くの知見を得ることができました。

現在のバージョンはミニマムな機能のみを搭載していますが、ユーザーのフィードバックを反映し、より役立つ機能を追加していく予定です。
将来的には、「家庭菜園のアプリといえばコレ！」と言われるような決定版を目指し、開発を進めていきます！

もし家庭菜園に興味のある方や、一緒にアプリ開発に取り組みたい方がいらっしゃったらぜひご連絡いただきたいです🌱
