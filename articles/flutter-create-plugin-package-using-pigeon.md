---
title: "【Flutter】Pigeon を使って Plugin Package を作成する"
emoji: "🐦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["firebase", "dart", "pigeon", "plugin", "package"]
published: true
publication_name: "altiveinc"
---

## はじめに

こんにちは！Altive株式会社のFlutterアプリ開発者の小林遼太（[@naipaka](https://twitter.com/naipakapaka)）です🦙

今月初めに、弊社で開発した「めでた！」という家庭菜園をサポートするアプリをリリースしました。

https://apps.apple.com/app/id6479965558

https://play.google.com/store/apps/details?id=jp.co.altive.medeta

このアプリの機能の一つに [複数枚の画像からタイムラプス動画を作成する](https://x.com/medetaapp/status/1777557089340317799) という機能があるのですが、処理自体は各ネイティブ側で行なっており、この機能を他のアプリでも流用できるように Plugin Package として切り出しています。（パッケージの公開はしていません）

このとき初めて [Pigeon](https://pub.dev/packages/pigeon) を使った Plugin Package を作成したので、その手順をまとめました。

## Pigeon とは

Flutter とプラットフォーム間の通信を行うツールとして、[MethodChannel](https://docs.flutter.dev/platform-integration/platform-channels) がありますが、プラットフォーム間でのデータのやり取りが文字列で行われるため、型安全ではありません。

[Pigeon](https://pub.dev/packages/pigeon) は、Flutter とプラットフォーム間の通信を型安全で行うためのコード生成ツールです。

Dart 側でメソッドを定義し、インターフェースを自動生成することで、型安全な通信を行うことができるようになっています。

https://pub.dev/packages/pigeon

## Pigeon を使った Plugin Package の作成

それでは、タイムラプス動画を作成するパッケージを題材に、Pigeon を使った Plugin Package の作成手順を説明していきます。
※実際にタイムラプス動画を作成する実装は省略しています。

### 1. Plugin Package の作成 & Pigeon 追加

まずは、Plugin Package を作成していきます。
タイムラプスを作成するための Plugin Package として、`timelapse_creator` という名前で作成します。

```bash
flutter create --org jp.co.altive --template=plugin --platforms=android,ios timelapse_creator --project-name timelapse_creator
```

自動で作成された以下のファイルは MethodChannel を使った通信を行うためのファイルです。
Pigeon を使った実装では不要なため、削除しておきます。

- `lib/timelapse_creator_method_channel.dart`
- `lib/timelapse_creator_platform_interface.dart`

以下は、後ほど Barrel file として使うので、中身を空にしておきます。

- `lib/timelapse_creator.dart`

次に、Pigeon を依存関係に追加します。

```yaml:pubspec.yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  pigeon: ^18.0.0
```

これで、Pigeon を使った Plugin Package の作成準備が整いました。

### 2. Pigeon でインターフェースを定義

Pigeon を使って Flutter からプラットフォームの処理を呼び出すためのインターフェースを定義します。

プロジェクトのルートディレクトリに `pigeons` ディレクトリを作成し、`messages.dart` を作成します。

:::message
ディレクトリ名、ファイル名は任意のようですが、Pigeon の慣習として `pigeons` ディレクトリに `messages.dart` を作成することが多いようです。

参考：
- [pigeon_plugin_example](https://github.com/gaaclarke/pigeon_plugin_example/blob/master/pigeons/messages.dart)
- [video_player](https://github.com/flutter/packages/blob/main/packages/video_player/video_player_avfoundation/pigeons/messages.dart)
:::

`messages.dart` に以下のようにインターフェースを定義します。

```dart:message.dart
import 'package:pigeon/pigeon.dart';

// 1. PigeonOptionsを使って、各プラットフォームの出力先を指定。
@ConfigurePigeon(
  PigeonOptions(
    dartOut: 'lib/src/messages.dart',
    dartOptions: DartOptions(),
    kotlinOut:
        'android/src/main/kotlin/jp/co/altive/timelapse_creator/Messages.kt',
    kotlinOptions: KotlinOptions(),
    swiftOut: 'ios/Classes/Messages.swift',
    swiftOptions: SwiftOptions(),
  ),
)
// 2. Input と Output のクラスを定義。
class SourceMessage {
  SourceMessage(this.imageBytesList, this.frameRate);

  List<Uint8List?> imageBytesList;
  int frameRate;
}

class TimelapseMessage {
  TimelapseMessage(this.videoBytes);

  Uint8List videoBytes;
}

// 3. インターフェースを定義。
@HostApi()
// ignore: one_member_abstracts
abstract class TimelapseCreatorApi {
  @async
  TimelapseMessage create(SourceMessage msg);
}
```

#### ⅰ. PigeonOptionsを使って、各プラットフォームの出力先を指定

`@ConfigurePigeon` の引数に `PigeonOptions` を指定することで、各プラットフォームの出力先を指定できます。
今回は、ネイティブ側の処理を Swift と Kotlin で行うため、それぞれの出力先を指定しています。

#### ⅱ. Input と Output のクラスを定義

`SourceMessage` と `TimelapseMessage` は、それぞれ入力と出力のデータを定義しています。
ここでは、複数の画像データを受け取り、タイムラプス動画を出力するためのデータを定義しています。

サポートされているデータ型は以下の通りです。
https://docs.flutter.dev/platform-integration/platform-channels#codec

#### ⅲ. インターフェースを定義

`@HostApi` を付与した `TimelapseCreatorApi` は、Dart 側で呼び出すメソッドを定義しています。

今回のタイムラプス生成のように非同期で処理を行う場合は、`@async` を付与する必要があります。

---

インターフェースが定義できたら、Pigeon を使ってコードを生成します。

```bash
flutter pub run pigeon --input pigeons/messages.dart
```

これで、各プラットフォームのコードが生成されます。

- `lib/src/messages.dart`
- `android/src/main/kotlin/jp/co/altive/timelapse_creator/Messages.kt`
- `ios/Classes/Messages.swift`

### 3. 各プラットフォームでインターフェースを実装

最後に、各プラットフォームでインターフェースを実装します。

#### iOS

パッケージ作成時に生成された `ios/Classes/TimelapseCreatorPlugin.swift` を以下のように修正します。

```diff swift:TimelapseCreatorPlugin.swift
import Flutter
import UIKit

- public class TimelapseCreatorPlugin: NSObject, FlutterPlugin {
+ public class TimelapseCreatorPlugin: NSObject, FlutterPlugin, TimelapseCreatorApi {
  public static func register(with registrar: FlutterPluginRegistrar) {
-     let channel = FlutterMethodChannel(name: "timelapse_creator", binaryMessenger: registrar.messenger())
-     let instance = TimelapseCreatorPlugin()
-     registrar.addMethodCallDelegate(instance, channel: channel)
+     // 1. 自動生成された TimelapseCreatorApiSetup を使って、インスタンスを登録。
+     let binaryMessenger = registrar.messenger()
+     let instance = TimelapseCreatorPlugin()
+     TimelapseCreatorApiSetup.setUp(binaryMessenger: binaryMessenger, api: instance)
  }

-   public func handle(_ call: FlutterMethodCall, result: @escaping FlutterResult) {
-     switch call.method {
-     case "getPlatformVersion":
-       result("iOS " + UIDevice.current.systemVersion)
-     default:
-       result(FlutterMethodNotImplemented)
-     }
-   }
+   func create(msg: SourceMessage, completion: @escaping (Result<TimelapseMessage, Error>) -> Void) {
+         // 2. ここで画像からタイムラプス動画へ変換する処理を実装。
+   }
}
```

##### ⅰ. 自動生成された TimelapseCreatorApiSetup を使って、インスタンスを登録

`TimelapseCreatorPlugin` で `TimelapseCreatorApi` インターフェースを実装し、`TimelapseCreatorApiSetup.setUp` を使ってインスタンスを登録します。

これにより、Dart 側から `TimelapseCreatorApi` のメソッドを呼び出すことができるようになります。

##### ⅱ. ここで画像からタイムラプス動画へ変換する処理を実装

`create` メソッドは、`TimelapseCreatorApi` で自動生成されたインターフェースのメソッドです。

ここで、画像からタイムラプス動画へ変換する処理を実装します。

:::message
Xcode で修正する場合は、以下の手順で行っていました。
他にいいやり方あれば教えていただきたいです🙏
1. `example/ios` で pod install
2. `example/ios/Runner.xcworkspace` を開く
3. `Pods/Development Pods/timelapse_creator`の中を辿り、`Classes/TimelapseCreatorPlugin.swift` を開く
![Xcode](/images/pigeon-edit-in-xcode.png)
*※`Pods/Development Pods/timelapse_creator` は、プロジェクトによって異なります。*
:::

#### Android

パッケージ作成時に生成された `android/src/main/kotlin/jp/co/altive/timelapse_creator/TimelapseCreatorPlugin.kt` を以下のように修正します。

```diff kotlin:TimelapseCreatorPlugin.kt
package jp.co.altive.timelapse_creator

- import androidx.annotation.NonNull

+ import SourceMessage
+ import TimelapseCreatorApi
+ import TimelapseMessage
import io.flutter.embedding.engine.plugins.FlutterPlugin
- import io.flutter.plugin.common.MethodCall
- import io.flutter.plugin.common.MethodChannel
- import io.flutter.plugin.common.MethodChannel.MethodCallHandler
- import io.flutter.plugin.common.MethodChannel.Result

/** TimelapseCreatorPlugin */
- class TimelapseCreatorPlugin: FlutterPlugin, MethodCallHandler {
-   /// The MethodChannel that will the communication between Flutter and native Android
-   ///
-   /// This local reference serves to register the plugin with the Flutter Engine and unregister it
-   /// when the Flutter Engine is detached from the Activity
-   private lateinit var channel : MethodChannel
+ class TimelapseCreatorPlugin : FlutterPlugin, TimelapseCreatorApi {

  override fun onAttachedToEngine(flutterPluginBinding: FlutterPlugin.FlutterPluginBinding) {
-     channel = MethodChannel(flutterPluginBinding.binaryMessenger, "timelapse_creator")
-     channel.setMethodCallHandler(this)
+     // 1. 自動生成された TimelapseCreatorApi.setUp を使って、バイナリメッセンジャーと自身を登録
+     TimelapseCreatorApi.setUp(flutterPluginBinding.binaryMessenger, this)
  }

+  override fun create(msg: SourceMessage, callback: (Result<TimelapseMessage>) -> Unit) {
+    // 2. ここで画像からタイムラプス動画へ変換する処理を実装。
+  }

-   override fun onMethodCall(call: MethodCall, result: Result) {
-     if (call.method == "getPlatformVersion") {
-       result.success("Android ${android.os.Build.VERSION.RELEASE}")
-     } else {
-       result.notImplemented()
-     }
-   }

  override fun onDetachedFromEngine(binding: FlutterPlugin.FlutterPluginBinding) {
-     channel.setMethodCallHandler(null)
  }
}
```

##### ⅰ. 自動生成された TimelapseCreatorApi.setUp を使って、バイナリメッセンジャーと自身を登録

`TimelapseCreatorPlugin` で `TimelapseCreatorApi` インターフェースを実装し、`TimelapseCreatorApi.setUp` を使ってバイナリメッセンジャーと自身を登録します。

これにより、Dart 側から `TimelapseCreatorApi` のメソッドを呼び出すことができるようになります。

##### ⅱ. ここで画像からタイムラプス動画へ変換する処理を実装

Swift同様、`create`メソッドは、`TimelapseCreatorApi` で自動生成されたインターフェースのメソッドです。

ここで、画像からタイムラプス動画へ変換する処理を実装します。

:::message
こちらは、Android Studio で修正するのがおすすめです。
`timelapse_creator/android` を Android Studio で開いて、`TimelapseCreatorPlugin.kt` を開いてください。
![Android Studio](/images/pigeon-edit-in-android-studio.png)
:::

### 4. Dart 側で API を呼び出す

最後に、Dart 側で API を呼び出す処理を実装します。

`lib/src` ディレクトリに `timelapse_creator.dart` を作成し、以下のように実装します。

```dart:lib/src/timelapse_creator.dart
import 'package:flutter/foundation.dart';

import 'messages.dart';

class TimelapseCreator {
  final _api = TimelapseCreatorApi();

  Future<Uint8List> create({
    required List<Uint8List> imageBytesList,
    required int frameRate,
  }) async {
    final response = await _api.create(
      SourceMessage(imageBytesList: imageBytesList, frameRate: frameRate),
    );
    return response.videoBytes;
  }
}
```

そして、Barrel file として、`lib/timelapse_creator.dart` に以下のように追記します。

```dart:lib/timelapse_creator.dart
export 'src/timelapse_creator.dart';
```

これで、このパッケージを依存関係に追加すれば、Flutter 側からプラットフォームの処理を呼び出すことができるようになりました！

```dart:example
final timelapseCreator = TimelapseCreator();
final videoBytes = await timelapseCreator.create(
  imageBytesList: imageBytesList,
  frameRate: frameRate,
);
```

## 最後に

機会がないとあまり触れることのない Pigeon ですが、型安全な通信を行うためにはとても便利なツールでした。

flutterfire プラグインで導入されているのを見てからずっと気になっていたので、今回使えてよかったです。

Pigeon を使って Plugin Package を作成する際は、ぜひ参考にしてみてください🐦
