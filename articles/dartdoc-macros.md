---
title: "【Flutter/Dart】dartdoc の Macros を使ってドキュメントコメントを再利用する"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [flutter,dart,dartdoc,customlint,lint]
published: false
---

本記事は、[Flutter Advent Calendar 2024](https://qiita.com/advent-calendar/2024/flutter) の 15 日目の記事です。

## はじめに

こんにちは！オルティブ株式会社の Flutter アプリ開発者の小林遼太（[@naipaka](https://twitter.com/naipakapaka)）です🦙

本記事では [dartdoc](https://pub.dev/packages/dartdoc) の [Macros](https://pub.dev/packages/dartdoc#macros) について紹介します。少しニッチなお話です。

:::message
現在 Experimental な機能として提供されている [Dart macro system](https://dart.dev/language/macros) の話ではありません🙏
:::

みなさんは、Dartのドキュメント生成ツールである `dartdoc` の `Macros` を使ったことがありますか？
下記のようなドキュメントコメントです。

```dart
/// {@template template_name}
/// Some shared docs
/// {@endtemplate}
```

```dart
/// Some comment
/// {@macro template_name}
/// More comments
```

`dartdoc` の `Macros` について知らなかった方や、見たことはあるけど使ったことがない方に向けて、その概要や活用場面を紹介します。

また、`dartdoc` の `Macros` を使ったドキュメントコメントを生成するアシスト機能を Custom Lint で作成したのでそちらも合わせて紹介します！

## `dartdoc` の `Macros` とは

https://pub.dev/packages/dartdoc#macros

`dartdoc` の `Macros` は、ドキュメントコメント内で再利用可能なテンプレートを定義する機能です。

先ほども紹介したように、`{@template}` と `{@endtemplate}` で囲まれた部分がテンプレートとして定義され、`{@macro}` でそのテンプレートを呼び出すことができます。

```dart
/// {@template template_name}
/// Some shared docs
/// {@endtemplate}
```

```dart
/// Some comment
/// {@macro template_name}
/// More comments
```

注意点として、テンプレートの定義はスコープが全体に適用されます。

`dartdoc` がテンプレートを含むファイルを読み取ると、
そのテンプレートが `dartdoc` が現在処理しているすべてのドキュメントで使用可能になります。

この仕様により、異なるパッケージで `dartdoc` を実行された場合、テンプレートが衝突する可能性があります。
そのため、テンプレート名にはパッケージ名やライブラリ名を含めるなど、衝突を避ける工夫が推奨されています。

利用されている例を見ると、テンプレート名は `.` で区切られた名前空間を持つことが多いようです。

https://github.com/flutter/flutter/blob/c9477b4306a1ea9ef48bb1cb21f2b9c3e8d19622/packages/flutter/lib/src/widgets/pages.dart#L33-L40

https://github.com/rrousselGit/riverpod/blob/288c9b53e308f902f6309eba1c173bf745e0cfaa/packages/hooks_riverpod/lib/src/consumer.dart#L5-L17

## `dartdoc` の `Macros` の活用例

主に公開パッケージのドキュメントで活用されているようですが、
アプリ開発内での活用例としては、以下のような場面が考えられます。

### 1. 同じコメントを複数のファイルで使い回す

「同じコメントを使う箇所なんてそんなにあるだろうか？」と思われるかもしれませんが、
規模が大きくなってきたりすると、意外と使い回しできるコメントがあるかと思います。

私が参画しているプロジェクトでは、以下のようなコメントが複数のファイルで繰り返し使われていたり、
他にも使い回しが可能なコメントが意外とありました。

```dart
/// xxx。 
///
/// ルート設定していないため `Navigator.push` を使用して遷移させる。
class XxxView extends StatelessWidget {
```

皆さんも、自身のプロジェクトで探してみると意外と使い回しできるコメントがあるかもしれません！
少しでもコメントを共通化することで、ドキュメントの一貫性を保つことができます。

### 2. クラスの説明とコンストラクタの説明を同一にする

もう一つの活用例としては、クラスの説明とコンストラクタの説明を同一にすることです。

Lint ルール で [public_member_api_docs](https://dart.dev/tools/linter-rules/public_member_api_docs) を有効にしている場合、
コンストラクタにも必ずドキュメントコメントを記載する必要があります。

これまで私は、コンストラクタの説明に `/// [Xxx] インスタンスを作成。` というようなドキュメントコメントを書いていましたが、
考えてみるとコンストラクタがインスタンスを作成するのは自明で冗長です。

また、IDE 上で利用側からドキュメントコメントを参照した時に見たいのは、コンストラクタの説明よりもクラスの説明であると思いました。

そこで、下記のように書くことで、冗長な説明を避けつつ、クラスの説明とコンストラクタの説明を同一にすることができます。
また、IDE 上で利用側からドキュメントコメントを参照したい時にも意味のあるコメントが表示され、利用側でコードを理解しやすくなります。

```dart
/// {@template user_description}
/// Represents a user with an ID and a name.
/// {@endtemplate}
class User {
  /// {@macro user_description}
  const User(this.id, this.name);

  /// {@macro user_description}
  ///
  /// Use this constructor to create a guest user.
  const User.guest()
      : id = 0,
        name = 'Guest';

  final int id;
  final String name;
}
```

**例**

| -           | Before                                           | After                                          |
| ----------- | ------------------------------------------------ | ---------------------------------------------- |
| Doc Comment | ![before](/images/dartdoc-macros-before.png)     | ![after](/images/dartdoc-macros-after.png)     |
| 利用側      | ![before](/images/dartdoc-macros-use-before.png) | ![after](/images/dartdoc-macros-use-after.png) |

Riverpod でも同様の活用例が見られます。（目的はもしかしたら違うかもしれません💭）

https://github.com/rrousselGit/riverpod/blob/288c9b53e308f902f6309eba1c173bf745e0cfaa/packages/hooks_riverpod/lib/src/consumer.dart#L5-L17

## `dartdoc` の `Macros` を使った doc comment を生成するアシスト機能

少しでも手間を減らすために、`dartdoc` の `Macros` を使った doc comment を生成するアシスト機能を作成しました！

以下3つの機能を [altive_lints](https://pub.dev/packages/altive_lints) に追加しています。気になる方はぜひ使ってみてください！

### 1. Add macro template documentation comment

クラス宣言に Macros テンプレートを追加します。
クラス宣言にカーソルを置いて「Add macro template documentation comment」実行すると、ドキュメントが作成されます。

**Before:**

```dart
class MyClass {
  // クラスの実装
}
```

**アシスト適用後:**

```dart
/// {@template my_package.MyClass}
///
/// {@endtemplate}
class MyClass {
  // クラスの実装
}
```

### 2. Add macro documentation comment

クラスのコンストラクタ宣言とメソッド宣言に Macros コメントを追加します。
コンストラクタ宣言やメソッド宣言にカーソルを置いて「Add macro documentation comment」実行すると、ドキュメントが作成されます。

**Before:**

```dart
void myFunction() {
  // Function implementation
}
```

**アシスト適用後:**

```dart
/// {@macro my_package.myFunction}
void myFunction() {
  // Function implementation
}
```


### 3. Wrap with macro template documentation comment

すでに存在するドキュメントコメントを Macros テンプレートで囲みます。
ドキュメントコメントを選択して「Wrap with macro template documentation comment」実行すると、ドキュメントが作成されます。

**Before:**

```dart
/// Some comment
/// More comments
class MyClass {
  // クラスの実装
}
```

**アシスト適用後:**

```dart
/// {@template my_package.MyClass}
/// Some comment
/// More comments
/// {@endtemplate}
class MyClass {
  // クラスの実装
}
```

## まとめ

本記事では、`dartdoc` の `Macros` の概要や活用例について紹介しました。

ドキュメントコメント内で再利用可能なテンプレートを定義して、日頃からドキュメントを整備していきたいですね。

もし他の活用方法などあれば、ぜひ教えていただきたいです！

最後までお読みいただき、ありがとうございました🙇‍♂️
