---
title: "Flutterアプリから送信したFirebase AnalyticsのデータをBigQueryにエクスポートして解析する"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["firebase", "bigquery", "analytics", "flutter"]
published: false
publication_name: "altiveinc"
---

## はじめに

こんにちは！Altive株式会社のFlutterアプリ開発者の小林遼太（[@naipaka](https://twitter.com/naipakapaka)）です🦙

みなさんはアプリの利用状況を把握するためにどのような方法を使っていますか？

これまでFlutterでアプリを開発していて、Firebase Analyticsを使ってアプリの利用状況を確認することはあったのですが、
それだけだとアプリの利用状況の詳細を把握するのに不十分で、クライアントの要望に応えることができないことがありました。

Firebase Analyticsへ送信したデータをBigQueryにエクスポートして解析できるようにしたことでアプリの利用状況をより詳細に把握することができるようになったので、
今回はFlutterアプリから送信したFirebase AnalyticsのデータをBigQueryにエクスポートして解析するまでの手順をまとめました。

## イベント送信

送信するイベントや解析したい情報はアプリによって異なると思いますが、デフォルトで送信されているイベント以外に下記を要最低限送信するようにしています。

- ユーザーID
- ユーザープロパティ
- 画面遷移

### ユーザーID

ユーザーIDを設定することで、ユーザーごとのイベントを把握できます。
任意の操作を行なったユーザー一覧を元にキャンペーンを行ったり、ユーザーごとの行動を把握して改善に活かせます。

```dart
    ref.listen(userChangesProvider.future, (_, asyncUser) async {
      // ユーザーの認証状態に変更があったらユーザーIDを設定する。
      final user = await asyncUser;
      if (user == null) {
        await _analytics.setUserId();
      } else {
        await _analytics.setUserId(id: userId);
      }
    });
```

### ユーザープロパティ

ユーザーに紐づくプロパティを設定することで、属性毎に解析することができます。
例えばプロフィール登録があるアプリなどでプロフィール画像の登録状況や登録日などを設定すると、属性毎の維持率や新規機能の利用状況などを把握することができます。

```dart
  /// ユーザーのプロパティを設定する。
  Future<void> setUserProperties(Map<String, String?> properties) async {
    for (final property in properties.entries) {
      await _analytics.setUserProperty(
        name: property.key,
        value: property.value,
      );
    }
  }
```

### 画面遷移

GoRouterを使って画面遷移を管理している場合、`FirebaseAnalyticsObserver`を使って画面遷移を送信することができます。
どの詳細ページに遷移したかが把握できるようにパスを送信するのがおすすめです。

```dart
  return GoRouter(
    routes: $appRoutes,
    observers: [
      FirebaseAnalyticsObserver(
        analytics: _analytics,
        nameExtractor: (settings) {
          // e.g. /, /home, /user/1
          return settings.name ?? 'unknown';
        },
      ),
    ],
    initialLocation: SplashRouteData.path,
  );
```

## エクスポート設定

BigQueryにエクスポートするためには、Firebaseのコンソールからエクスポート設定を行う必要があります。
設定前のデータはエクスポートされないので、BigQueryで解析することを想定している場合はアプリリリース前に設定しておくことをおすすめします。

1. 「プロジェクトの設定」>「統合」>「BigQuery」の「リンク」
![](/images/bigquey-export-1.png)
2. 「①FirebaseとBigQueryのリンクについて」で「次へ」
![](/images/bigquey-export-2.png)
3. 「②統合を構成する」で「BigQueryにリンク」
![](/images/bigquey-export-3.png)
4. 「Google Analytics」のスイッチをクリック
![](/images/bigquey-export-4.png)
5. 「BigQueryにエクスポート」をクリック
![](/images/bigquey-export-5.png)
6. 「エクスポート設定」で「毎日」と「エクスポートに広告IDを含める」にチェック
![](/images/bigquey-export-6.png)

これでBigQueryにエクスポートする設定は完了です。
翌日の朝にはエクスポートされているはずです！
（自分の環境だと朝の5時から6時くらいにエクスポートされていました）

## 解析

GCPの[BigQuery](https://console.cloud.google.com/bigquery)でクエリを書くことでエクスポートしたデータを元に自由に解析することができます。

例えば、`search`というイベントを送信したユーザーの各属性を把握したい場合は下記のようなクエリを書くことができます。

```sql
SELECT
  event_name,
  user_id,
  -- イベントパラメータで計測しているパラメータの値
  MAX(IF(event_params.key = 'search_word', event_params.value.string_value, NULL)) as search_word,
  -- ユーザープロパティで計測しているパラメータの値
  MAX(IF(user_properties.key = 'is_profile_image_set', user_properties.value.string_value, NULL)) as is_profile_image_set,
  MAX(IF(user_properties.key = 'registration_date', user_properties.value.string_value, NULL)) as registration_date,
  MAX(IF(user_properties.key = 'age', user_properties.value.string_value, NULL)) as age,
  MAX(IF(user_properties.key = 'gender', user_properties.value.string_value, NULL)) as gender,
FROM
    -- プロジェクトID.データセットID.テーブルID
    `xxxxx.analytics_xxxxx.events_*`,
    UNNEST(event_params) as event_params,
    UNNEST(user_properties) as user_properties
WHERE
    -- テーブルのサフィックスを指定して、過去1週間分のデータを取得
    _table_suffix BETWEEN '20240101' AND '20240107'
    AND event_name = 'search'
```

クエリの結果はCSVやJSONなどでダウンロードすることができます。

https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax

:::message
クエリを共有して使うためにプロジェクトクエリとして保存したい場合は「BigQuery Admin」の権限が必要なので注意が必要です。
https://cloud.google.com/bigquery/docs/access-control?hl=ja#bigquery.admin

（こちらの情報は[@RyosukeOK](https://zenn.dev/ryosuke_ok)さんに教えていただきました🙇‍♂️）
:::

## BigQueryの料金

最後にBigQueryの料金についてもまとめます。
（以下オンデマンド料金の場合です）

### 分析（クエリ）料金

- クエリによって処理されたデータのバイト数に基づいて課金される
- 東京リージョン（asia-northeast）の場合、$6.00 per TB、毎月 1 TB まで無料

クエリの実行前に、実行するクエリのバイト数を確認することができます。
取得するデータの期間を狭めたり、必要なカラムのみ取得するようにすることで、クエリのバイト数を抑えることができます。
たまに解析するくらいであれば無料枠で十分に収まりそうです。
![](/images/bigquery-query-byte.png)

### ストレージ料金

#### アクティブストレージ：

- 対象: 90 日間以内に追加・編集されたテーブル
- 東京リージョン（asia-northeast）の場合、$0.020 per GB、毎月 10 GB まで無料

#### 長期保存：

- 対象: 90 日間連続して編集されていないテーブル
- 東京リージョン（asia-northeast）の場合、$0.010 per GB、毎月、10 GB まで無料
- 長く使うにつれて費用がかかってくる

https://cloud.google.com/bigquery/pricing?hl=ja

## まとめ

Firebase Analyticsで送信したデータをBigQueryにエクスポートして解析することで、アプリの利用状況をより詳細に把握することができるようになりました。

BigQueryの料金は激しく使用しなければ無料枠で十分に収まりそうなので、積極的に活用していきたいと思います！
