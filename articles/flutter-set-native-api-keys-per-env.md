---
title: "【Flutter】環境ごとのAPIキーをiOS/Androidネイティブ側に設定する【Google Maps API】"
emoji: "🔑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "googlemap", "ios", "android", "apikey"]
published: true
publication_name: "altiveinc"
---

## はじめに

こんにちは！Altive株式会社のFlutterアプリ開発者の小林遼太（[@naipaka](https://twitter.com/naipakapaka)）です🦙

FlutterでGoogle Mapsを使う際には、APIキーを各プラットフォーム（iOS/Android）ごとに設定する必要がありますが、
APIキーが`dev`, `stg`, `prod`など環境ごとに分かれている場合、各ネイティブに対して環境ごとに異なるAPIキーを渡してあげる必要があります。

今回はGoogle Mapsを使う際に環境ごとのAPIキーをiOS/Androidネイティブで設定する機会があったので、備忘録として対応の手順をまとめました。

## 環境分け

今回は`dart-define-from-file`を使った環境分けを前提としています。

環境分けの詳細については、下記記事を参照してください！

https://zenn.dev/altiveinc/articles/separating-environments-in-flutter

各環境ごとの定義ファイルは下記のとおりです。
Google Maps の API キーは iOS と Android で異なるため、それぞれ`androidGoogleMapApiKey`と`iOSGoogleMapApiKey`を定義しています。
また、今回は`appName`や`appId`は省略しています。

```env:dart_defines/dev.env
flavor="dev"
...（省略）
androidGoogleMapApiKey="xxx"
iOSGoogleMapApiKey="xxx"
```

```env:dart_defines/prod.env
flavor="prod"
...（省略）
androidGoogleMapApiKey="xxx"
iOSGoogleMapApiKey="xxx"
```

## Android 側に API キーを設定する

Android アプリに Google Maps API キーを設定するには、`AndroidManifest.xml`に`meta-data`タグを追加する必要があります。

```xml:android/app/src/main/AndroidManifest.xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.google_maps_in_flutter">
        ...
        <meta-data android:name="com.google.android.geo.API_KEY" android:value="YOUR-KEY-HERE"/>
        ...
</manifest>
```

ここの値を環境ごとに分けるために、`build.gradle`で`dart-define`から渡された値を取得して`AndroidManifest.xml`で参照できるようにします。

以下のように、`resValue`を使って`AndroidManifest.xml`に渡す文字列リソースを定義します。

```gradle:android/app/build.gradle
// Dart-Defines.
def dartDefines = [:];
if (project.hasProperty('dart-defines')) {
    dartDefines = dartDefines + project.property('dart-defines')
        .split(',')
        .collectEntries { entry ->
            def pair = new String(entry.decodeBase64(), 'UTF-8').split('=')
            [(pair.first()): pair.last()]
        }
}
...
android {
    ...
    defaultConfig {
        minSdkVersion flutter.minSdkVersion
        targetSdkVersion flutter.targetSdkVersion
        versionCode flutterVersionCode.toInteger()
        versionName flutterVersionName
        resValue "string", "google_map_api_key", "${dartDefines.androidGoogleMapApiKey}"
    }
    ...
}
```

これにより、`AndroidManifest.xml`で`@string/google_map_api_key`を参照することができるようになります。

```xml:android/app/src/main/AndroidManifest.xml
<meta-data android:name="com.google.android.geo.API_KEY" android:value="@string/google_map_api_key"/>
```

これで、環境ごとに異なる API キーを Android アプリに設定することができました！

## iOS 側に API キーを設定する

iOS アプリに Google Maps API キーを設定するには、`AppDelegate.swift`に API キーを設定する必要があります。

```swift:ios/Runner/AppDelegate.swift
@UIApplicationMain
@objc class AppDelegate: FlutterAppDelegate {
  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    GeneratedPluginRegistrant.register(with: self)

    // TODO: Add your Google Maps API key
    GMSServices.provideAPIKey("YOUR-API-KEY")

    return super.application(application, didFinishLaunchingWithOptions: launchOptions)
  }
}
```

ここの値を環境ごとに分けるためには、`Info.plist`に`dart-define`から渡した値を設定し、`AppDelegate.swift`で参照する形にします。

以下のように、`Info.plist`に Dictionary型の`Google Maps API Key`を追加し、`dart-define`から渡された値を設定します。

```xml:ios/Runner/Info.plist
<plist version="1.0">
    <dict>
        <key>Google Maps API Key</key>
        <string>${iOSGoogleMapApiKey}</string>
        ...
```

そして、`AppDelegate.swift`で`Info.plist`から API キーを取得し、`GMSServices.provideAPIKey`に渡します。

```swift:ios/Runner/AppDelegate.swift
@UIApplicationMain
@objc class AppDelegate: FlutterAppDelegate {
  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    GeneratedPluginRegistrant.register(with: self)

    if let googleMapApiKey = Bundle.main.infoDictionary?[“Google Maps API Key”] as? String {
      GMSServices.provideAPIKey(googleMapApiKey)
    }

    return super.application(application, didFinishLaunchingWithOptions: launchOptions)
  }
}
```

これで、環境ごとに異なる API キーを iOS アプリに設定することができました！

## まとめ

実際にやってみると意外と簡単にネイティブ側に各環境ごとのAPIキーを設定することができますね✍️

今回はGoogle Maps APIキーを例に挙げましたが、他のAPIやSDKでも同様の方法で環境ごとに異なるAPIキーを設定することができるかと思います！

## 参考

https://zenn.dev/altiveinc/articles/separating-environments-in-flutter

https://medium.com/flutter-jp/flavor-b952f2d05b5d