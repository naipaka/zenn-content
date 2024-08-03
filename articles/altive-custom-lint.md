---
title: "【Flutter】altive_lints に追加したカスタム Lint ルールを紹介"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [flutter,dart,customlint, lint]
published: true
publication_name: "altiveinc"
---

## はじめに

こんにちは！オルティブ株式会社のFlutterアプリ開発者の小林遼太（[@naipaka](https://twitter.com/naipakapaka)）です🦙

弊社では、Flutterアプリ開発時に [altive_lints](https://pub.dev/packages/altive_lints) を使ってコードの品質を保つようにしています。
altive_lints では、すべてのルールを有効にする [all_lint_rules.yaml](https://github.com/altive/altive_lints/blob/main/packages/altive_lints/lib/all_lint_rules.yaml) と、その中からオルティブ推奨ルールを選択している [altive_lints.yaml](https://github.com/altive/altive_lints/blob/main/packages/altive_lints/lib/altive_lints.yaml) を提供しています。

先日、上記に加えて、弊社での開発スタイルに合わせたカスタム Lint ルールを追加しました。
目的は、レビューの工数の削減やコードの品質保持のためです。

今回はその追加したルールたちを紹介していきます！

Custom Lint の説明や Lint ルールの作成の仕方については、別の記事にまとめているのでこちらも合わせてみていただけると嬉しいです。

https://zenn.dev/altiveinc/articles/flutter-custom-lint-rule-creation

## 追加したカスタム Lint ルール

以下、altive_lints 1.11.1 で追加されているカスタム Lint ルールの一覧です。
アルファベット順に並べています。

各ルールのリンクは GitHub のソースコードに飛ぶようになっていますので、詳細な実装や設定方法を知りたい場合はそちらを参照してください。

---

### 1. [avoid_consecutive_sliver_to_box_adapter](https://github.com/altive/altive_lints/blob/main/packages/altive_lints/lib/src/lints/avoid_consecutive_sliver_to_box_adapter.dart)

`CustomScrollView` の `slivers` に `SliverToBoxAdapter` が連続して配置されている場合に警告を出すルールです。

`SliverToBoxAdapter` は viewport 外に配置されてもレイアウトされるため、複数使用するとパフォーマンスが低下する可能性があります。

また公式ドキュメントでも、複数の`SliverToBoxAdapter` を使用するよりも、`SliverList`、`SliverFixedExtentList`、`SliverPrototypeExtentList`、または`SliverGrid`を使用することが推奨されています。

> Rather than using multiple SliverToBoxAdapter widgets to display multiple box widgets in a CustomScrollView, consider using SliverList, SliverFixedExtentList, SliverPrototypeExtentList, or SliverGrid, which are more efficient because they instantiate only those children that are actually visible through the scroll view's viewport.

https://api.flutter.dev/flutter/widgets/SliverToBoxAdapter-class.html

参考：https://zenn.dev/3ta/articles/5a439a8f0c4b62

#### BAD:
```dart
CustomScrollView(
  slivers: <Widget>[
    SliverToBoxAdapter(child: Text('Item 1')), // Consecutive usage
    SliverToBoxAdapter(child: Text('Item 2')), // LINT
  ],
);
```

#### GOOD:
```dart
CustomScrollView(
  slivers: <Widget>[
    SliverList.list(
      children: [
        Text('Item 1')
        Text('Item 2')
      ],
    ),
  ],
);
```

### 2. [avoid_hardcoded_color](https://github.com/altive/altive_lints/blob/main/packages/altive_lints/lib/src/lints/avoid_hardcoded_color.dart)

コード内でのハードコーディングされた `Color` や `Colors` の使用を避け、`ColorScheme`、`ThemeExtension`、その他のテーマのシステムを使用して色を定義することを推奨するルールです。

主にライト/ダークモードなどのテーマを定義している場合に使用することを想定しており、背景色による文字色の切り替えなどで、テーマによって変更される色を誤ってハードコーディングされることがないようにしています。

#### BAD:
```dart
ColoredBox(
  color: Color(0xFF00FF00), // LINT
);
```

#### GOOD:
```dart
ColoredBox(
  color: Theme.of(context).colorScheme.primary,
);
```

### 3. [avoid_hardcoded_japanese](https://github.com/altive/altive_lints/blob/main/packages/altive_lints/lib/src/lints/avoid_hardcoded_japanese.dart)

コード内でハードコーディングされた日本語のテキスト文字列を避け、`AppLocalizations` などの国際化対応を使用してテキストを定義することを推奨するルールです。

主に多言語対応を行っている場合に使用することを想定しており、多言語対応漏れを防ぐために使用します。

開発内でしか使用せず多言語化する必要がない文言の場合などは、適宜 ignore コメントを追加しています。

#### BAD:
```dart
final message = 'こんにちは'; // LINT
print('エラーが発生しました'); // LINT
```

#### GOOD:
```dart
final message = AppLocalizations.of(context).hello;
print(AppLocalizations.of(context).errorOccurred);
```

### 4. [avoid_shrink_wrap_in_list_view](https://github.com/altive/altive_lints/blob/main/packages/altive_lints/lib/src/lints/avoid_shrink_wrap_in_list_view.dart)

`ListView` で `shrinkWrap` プロパティを使用することを避けるよう推奨します。

`shrinkWrap` は、`ListView` の高さをその内容に合わせるために使用されますが、`ListView` 内の要素が多い場合や動的に変わる場合にはパフォーマンスが低下する可能性があるため、代わりに、`CustomScrollView` と `SliverList` を使用することが推奨されています。

ただ例えば、ダイアログ内に高さの余裕がある場合などに縮小するために使用されるのは問題ないため、そのような場合は適宜 ignore コメントを追加しています。

#### BAD:
```dart
ListView(
  shrinkWrap: true, // LINT
  children: <Widget>[
    Text('Hello'),
    Text('World'),
  ],
);
```

#### GOOD:
```dart
CustomScrollView(
  slivers: <Widget>[
    SliverList.list(
      children: [
        Text('Hello'),
        Text('World'),
      ],
    ),
  ],
);
```

### 5. [avoid_single_child](https://github.com/altive/altive_lints/blob/main/packages/altive_lints/lib/src/lints/avoid_single_child.dart)

複数の要素を持つことを意図した `children` プロパティを1つの子要素だけで使用することを警告します。
対象の Widget には `Column`、`Row`、`Flex`、`Wrap`、`ListView`、および`SliverList`のウィジェットが含まれます。

このルールにより可読性が向上し、要素を削除した場合に `children` が不要になったことにも気づきやすくなります。

#### BAD:
```dart
Column(
  children: <Widget>[YourWidget()], // LINT
);
```

#### GOOD:
```dart
Center(child: YourWidget());
// or
Column(
  children: <Widget>[YourWidget1(), YourWidget2()],
);
```

### 6. [prefer_clock_now](https://github.com/altive/altive_lints/blob/main/packages/altive_lints/lib/src/lints/prefer_clock_now.dart)

`DateTime.now()` の代わりに `clock.now()` を使用することを推奨します。
これは、`DateTime.now()` を使用してのテストが困難なためです。

[clock](https://pub.dev/packages/clock) パッケージを使用し、`clock.now()` を使用しておくことで、`withClock` メソッドを使ってテスト時に日時を差し替えることができます。

#### BAD:
```dart
var now = DateTime.now(); // LINT
```

#### GOOD:
```dart
var now = clock.now(); // Using 'clock' package
```

### 7. [prefer_dedicated_media_query_methods](https://github.com/altive/altive_lints/blob/main/packages/altive_lints/lib/src/lints/prefer_dedicated_media_query_methods.dart)

`MediaQuery.of` や `MediaQuery.maybeOf` の代わりに専用の `MediaQuery` メソッドを使用することを推奨します。

これは、`MediaQuery.sizeOf` や `MediaQuery.viewInsetsOf` のような専用メソッドを使用することで、不要なウィジェットの再構築を減らし、パフォーマンスを向上させるためです。

#### BAD:
```dart
var size = MediaQuery.of(context).size; // LINT
var padding = MediaQuery.maybeOf(context)?.padding; // LINT
```

#### GOOD:
```dart
var size = MediaQuery.sizeOf(context);
var padding = MediaQuery.viewInsetsOf(context);
```

### 8. [prefer_sliver_prefix](https://github.com/altive/altive_lints/blob/main/packages/altive_lints/lib/src/lints/prefer_sliver_prefix.dart)

`Sliver` 型 Widget を返す Widget のクラス名に「Sliver」を接頭辞に付けることを推奨するルールです。

これにより、利用側で `Sliver` 型 Widget であることが一目でわかり、可読性と一貫性が向上します。

ただ、現状だと下記のようなクラス名でも警告が出てしまうため、今後ある程度の柔軟性を持たせるように改善する予定です。
例: `MyWidget.sliver()` , `MyExcellentViewWithSliver()` , `AltiveSliverWidget()` , `_SliverWidget()`

#### BAD:
```dart
class MyCustomList extends StatelessWidget { // LINT
  @override
  Widget build(BuildContext context) {
    return SliverList(...);
  }
}

```
#### GOOD:
```dart
class SliverMyCustomList extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return SliverList(...);
  }
}
```

### 9. [prefer_space_between_elements](https://github.com/altive/altive_lints/blob/main/packages/altive_lints/lib/src/lints/prefer_space_between_elements.dart)

このルールは、クラス定義内の間隔規則を強制し、コンストラクタとフィールドの間、およびコンストラクタと`build` メソッドの間に空行を要求するものです。

適切な間隔を設けることで、コードの可読性と整理が向上し、クラスの異なるセクションを視覚的に区別しやすくする目的があります。

#### BAD:
```dart
class MyWidget extends StatelessWidget {
  MyWidget(this.title);
  final String title; // LINT
  @override // LINT
  Widget build(BuildContext context) {
    return Text(title);
  }
}

```
#### GOOD:
```dart
class MyWidget extends StatelessWidget {
  MyWidget(this.title);

  final String title;

  @override
  Widget build(BuildContext context) {
    return Text(title);
  }
}
```

## さいごに

今回は、[altive_lints](https://pub.dev/packages/altive_lints) に追加したカスタム Lint ルールを紹介しました。

まだ追加したばかりなので、実際に運用してみて不具合や改善点があれば随時追加・修正していきます。
今後も開発スタイルに合わせてカスタム Lint ルールを追加していく予定なので、 [altive_lints](https://pub.dev/packages/altive_lints) をチェックしていただけると嬉しいです！

また、他の会社やプロジェクトでのカスタム Lint ルールの活用事例などあれば、ぜひ教えていただきたいです！

最後まで読んでいただき、ありがとうございました🙇‍♂️
