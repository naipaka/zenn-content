---
title: "【Flutter】Custom Lintを使って独自のLintルールを作成する"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [flutter,dart,customlint, lint]
published: true
published_at: 2023-12-17
publication_name: "altiveinc"
---

本記事は、[Flutter Advent Calendar 2023](https://qiita.com/advent-calendar/2023/flutter)の17日目の記事です。

## はじめに

こんにちは！Altive株式会社のFlutterアプリ開発者の小林遼太（[@naipaka](https://twitter.com/naipakapaka)）です🦙

弊社では、受託案件の開発を効率化するために[FlutterFireパッケージのラッパーパッケージ](https://github.com/altive/flutterfire_adapter)を作成し、利用しています。
このパッケージの開発の過程で、特定のコーディングパターンを強制するようなLintルールが必要だと感じることがありました。
特に、`StreamController`インスタンスを宣言した際に`close`メソッドの呼び出しを強制する[close_sinks](https://dart.dev/tools/linter-rules/close_sinks)のようなルールが欲しいと考えていました。

そこで、Dartの[custom_lint](https://pub.dev/packages/custom_lint)を使って独自のLintルールを作成してみることにしたので、この記事ではその作成過程を紹介します。

## 完成例

`Config`インスタンスを宣言した際に、`dispose`メソッドの呼び出しを強制するルールです。
クラスのフィールドと関数内のローカル変数に対してルールが適用されます。

![](/images/custom-lint-dispose-config.gif)
*`Config`インスタンスを宣言した際に、`dispose`メソッドの呼び出しを強制するルール*

## Custom Lintとは

https://pub.dev/packages/custom_lint

[custom_lint](https://pub.dev/packages/custom_lint)は、Dartプロジェクトの保守性を向上させるために、パッケージ作者や開発者が独自のLintルールを簡単に作成できるツールです。
これにより、DartのデフォルトLintルールを拡張し、特定のコーディングパターンや慣習に合わせたルールを定義することが可能になります。

特徴は以下の通りです。

- **コマンドラインツール**: CIでLintのリストを取得するためのコマンドラインを自分で書く必要がない。
- **簡易セットアップ**: アナライザーサーバーやエラー処理を扱う必要がなく、Lintの作成に集中できる。
- **デバッグ**: Dartデバッガーを使用してLintを検査し、ブレークポイントを設定可能。
- **ホットリスタート対応**: Linterプラグインのソースコードを更新すると、IDE/アナライザーサーバーを再起動せずに動的にリスタートされる。
- **`// ignore:` および `// ignore_for_file:` の内蔵サポート**: 特定の行やファイル全体のLint警告を無視するコメントに対応。
- **テスト**: `// expect_lint` を使用した組み込みテストメカニズムを提供。
- **print(...)と例外**: プラグインがエラーやデバッグメッセージを出力した場合、ログファイルを生成する。

## セットアップ

[custom_lintのUsage](https://pub.dev/packages/custom_lint#usage)セクションに従って、独自のLintルールを作成するためのセットアップを行います。

### 依存関係の追加

まずは、プロジェクトを作成し、[custom_lint_builder](https://pub.dev/packages/custom_lint_builder)を追加します。
今回はFlutterFireのラッパーパッケージに関するLintルールなので、`flutterfire_lint`という名前でプロジェクトを作成します。
下記は執筆時点での最新バージョンです。

```bash
flutter create -t package packages/flutterfire_lint --project-name flutterfire_lint 
```

```yaml:pubspec.yaml
name: flutterfire_lint
environment:
  sdk: ^3.0.0

dependencies:
  analyzer: ^6.2.0
  analyzer_plugin: ^0.11.3
  custom_lint_builder: ^0.5.7
```

### Lintルール定義ファイルの作成

プロジェクト内に`lib/flutterfire_lint.dart`ファイルを作成し、Lintルールを定義します。

```dart:lib/flutterfire_lint.dart
import 'package:custom_lint_builder/custom_lint_builder.dart';

// Lintルールのエントリポイント。
PluginBase createPlugin() => _FlutterFirePlugin();

class _FlutterFirePlugin extends PluginBase {
  // 独自の警告/情報/エラーをリストで定義する。
  @override
  List<LintRule> getLintRules(CustomLintConfigs configs) => [];
}
```

上記の`getLintRules`メソッドのリストに、`DartLintRule`を継承した独自のLintルールを定義したクラスを追加します。

## ルールの作成

それでは、Lintルールを作成していきます。

### ルールの定義

今回は、`flutterfire_configurator`パッケージの`Config`インスタンスを宣言した際に、`dispose`メソッドの呼び出しを強制するルールを作成したいので、`dispose_config`という名前でルールを定義し、`Config`インスタンスの宣言時に`dispose`メソッドの呼び出しを強制するメッセージを定義します。
Lintルールに引っかかった場合は、ここで定義したメッセージが表示されます。

```dart:lib/src/dispose_config.dart
class DisposeConfig extends DartLintRule {
  const DisposeConfig() : super(code: _code);

  static const _code = LintCode(
    name: 'dispose_config',
    problemMessage: "Undisposed instance of 'Config'.\n"
        "Try invoking 'dispose' in the function in which the "
        "'Config' was created.",
  );

  @override
  void run(
    CustomLintResolver resolver,
    ErrorReporter reporter,
    CustomLintContext context,
  ) {
    // TODO: 変数定義やメソッド呼び出しを検知し、条件に応じてLintを表示する。
  }
}
```

### 変数定義やローカル変数宣言を検知

次は、`run`メソッド内で、`Config`インスタンスの宣言を検知し、`dispose`メソッドの呼び出しを検知する処理を実装してきます。

参考として[close_sinks](https://dart.dev/tools/linter-rules/close_sinks)のソースコードを見てみると、`registerNodeProcessors`で`FieldDeclaration`（フィールドの宣言）と`VariableDeclarationStatement`（ローカル変数の宣言）のコールバックを登録していることがわかります。

https://github.com/dart-lang/linter/blob/main/lib/src/rules/close_sinks.dart#L81-L87

`CustomLintContext`から静的解析結果を監視できる`LintRuleNodeRegistry`を取得できるので、同じように`FieldDeclaration`と`VariableDeclarationStatement`のコールバックを登録します。

:::message
本記事では、`FieldDeclaration`のコールバックで`Config`インスタンスの宣言を検知し`dispose`メソッドの呼び出しを検知するまでを実装します。
`VariableDeclarationStatement`のコールバックで`dispose`メソッドの呼び出しを検知する処理については、[こちらのPull Request](https://github.com/altive/flutterfire_adapter/pull/33)を参考にしてください。
:::

```dart:lib/src/dispose_config.dart
@override
  void run(
    CustomLintResolver resolver,
    ErrorReporter reporter,
    CustomLintContext context,
  ) {
    // フィールド宣言を監視。
    context.registry.addFieldDeclaration((node) {});

    // ローカル変数宣言を監視。
    context.registry.addVariableDeclarationStatement((node) {});
  }
```

各nodeには、`FieldDeclaration`と`VariableDeclarationStatement`と呼ばれるASTノードが渡され、これらを使用してフィールド宣言とローカル変数宣言を検知します。

:::message
AST（抽象構文木）は、プログラムのソースコードを木構造で表現したものです。
これにより、コードの構造が階層的に整理され、解析や変換が容易になります。
Lintのようなツールは、ASTを使用してコードのパターンを識別し、特定のルールに基づいてエラーや警告を生成します。
:::

### フィールド宣言から`Config`インスタンスの宣言を検知

まずは、フィールド宣言から`Config`インスタンスの宣言を検知する処理を実装します。

`Config`インスタンスの宣言を検知するには、`FieldDeclaration`からインスタンスの型を取得し、`Config`インスタンスの型であることを確認する必要があります。

```dart:lib/src/dispose_config.dart
    context.registry.addFieldDeclaration((node) {
      // 1. フィールド宣言からインスタンスの型を取得。
      final targetType = node.fields.type?.type;
      if (targetType == null) {
        return;
      }
      // 2. TypeCheckerを使用してインスタンスの型がConfigであることを確認。
      final isConfig = const TypeChecker.fromName(
        'Config',
        packageName: 'flutterfire_configurator',
      ).isExactlyType(targetType);
      if (!isConfig) {
        return;
      }
      // 3. Configインスタンスの宣言を検知した場合、Lintを表示。
      reporter.reportErrorForNode(_code, node);
    });
```

#### 1. フィールド宣言からインスタンスの型を取得

まず、`FieldDeclaration`の`fields`プロパティを調べます。これにはフィールドの宣言が含まれており、例えば`late Config<int> _intConfig;`の場合、`fields`にはこの宣言が含まれます。

次に、この`VariableDeclarationList`からフィールドの型を示す`TypeAnnotation`を取得します。`TypeAnnotation`の`type`プロパティには、宣言されたフィールドの具体的な型が含まれています。たとえば、`late Config<int> _intConfig`の場合、`type`には`Config<int>`という型情報が格納されています。

この型情報を使用して、フィールドが特定の型（この例では`Config`型）であるかを検証します。

#### 2. TypeCheckerを使用してインスタンスの型がConfigであることを確認

`TypeChecker`を使用して、`DartType`が`Config`であることを確認します。
`TypeChecker`は、`custom_lint_core`で提供されているクラスで、特定の型をチェックするためのユーティリティです。
[riverpod_lint](https://github.com/rrousselGit/riverpod/blob/master/packages/riverpod_analyzer_utils/lib/src/riverpod_types.dart)のコードでも使用されているため、参考にしてみてください。

#### 3. Configインスタンスの宣言を検知した場合、Lintを表示

`Config`インスタンスの宣言を検知した場合、`reportErrorForNode`メソッドを使用して、`FieldDeclaration`のASTノードに対してLintを表示します。
これで、下記のような形で警告が表示されることが確認できました。
（`dispose`メソッドの呼び出しを検知する処理を実装していないため、`dispose`が定義されているのに警告が表示されているのは想定通りです）

![](/images/lint-detect-config-instance-declaration.png)
*`Config`インスタンスの宣言を検知した場合、Lintを表示*

### `Config`インスタンスの宣言から`dispose`メソッドの呼び出しを検知

次は、`Config`インスタンスの宣言から`dispose`メソッドの呼び出しを検知する処理を実装します。

参考として[close_sinks](https://dart.dev/tools/linter-rules/close_sinks)のソースコードを追ってみると、以下のような手順で`close`メソッドの呼び出しを検知していることがわかります。

1. `FieldDeclaration`の`node`から自身またはその祖先ノードの中で`CompilationUnit`（コードの一部分を構成する単位（たとえばファイル全体や一部のコードブロックなど）を表す）型のノードを探す
2. `CompilationUnit`の子ノードから`MethodDeclaration`（メソッドの宣言）型や`PrefixedIdentifier`（プレフィックス付きの識別子（たとえば変数や関数などの名前））型のノードなどを探す
3. 見つかったノード全てからフィールド宣言と同じフィールド名かつ`close`メソッドを呼び出しているかを確認し、もしフィールド宣言と同じフィールド名で`close`が一度も呼び出されていない場合は、`FieldDeclaration`のASTノードに対してLintを表示する

今回は、`MethodDeclaration`型のノードを一通り探すだけでやりたいことはできそうなので、`MethodDeclaration`型のノードを探して、`dispose`メソッドの呼び出しを検知する処理を実装します。

```dart:lib/src/dispose_config.dart
  // `MethodDeclaration`型のノードを探す。
  Iterable<MethodInvocation> traverseMethodInvocations(AstNode node) {
    final methodInvocations = <MethodInvocation>[];
    void visitNode(AstNode node) {
      if (node is MethodInvocation) {
        methodInvocations.add(node);
      }
      if (node is FunctionExpressionInvocation) {
        final function = node.function;
        if (function is MethodInvocation) {
          methodInvocations.add(function);
        }
      }
      node.childEntities.whereType<AstNode>().forEach(visitNode);
    }

    visitNode(node);
    return methodInvocations;
  }
  // ...
    context.registry.addFieldDeclaration((node) {
      // ...
      if (!isConfig) {
        return;
      }
      // 自身またはその祖先ノードの中で`CompilationUnit`型のノードを探す。
      // `CompilationUnit`はコードの一部分を構成する単位（たとえばファイル全体や一部のコードブロックなど）を表す
      final compilationUnit = node.thisOrAncestorOfType<CompilationUnit>();
      if (compilationUnit == null) {
        return;
      }
      // `compilationUnit`の中で`MethodInvocation`型のノードを探す。
      final methodInvocations = traverseMethodInvocations(compilationUnit);

      // 1. `node.fields.variables`からフィールド名を取得。
      for (final field in node.fields.variables) {
        var isDisposed = false;

        // 2. `methodInvocations`からメソッド宣言一覧を取得、
        for (final methodInvocation in methodInvocations) {
          // 3. `methodInvocation.realTarget`からメソッドの実行対象を取得。
          final target = methodInvocation.realTarget;
          if (target is! SimpleIdentifier) {
            continue;
          }
          final methodName = methodInvocation.methodName.name;
          // 4. メソッドの実行対象がフィールド名と一致し、メソッド名が`dispose`であることを確認。
          if (target.name == field.name.lexeme && methodName == 'dispose') {
            isDisposed = true;
            break;
          }
        }

        if (!isDisposed) {
          // 5. フィールド宣言と同じフィールド名で`dispose`が一度も呼び出されていない場合は、Lintを表示。
          reporter.reportErrorForNode(_code, field);
        }
      }
    });
```

実装内容を説明していきます。

#### 1. `node.fields.variables`からフィールド名を取得

`node.fields.variables`からフィールド名を取得します。
`node.fields.variables`には、フィールド宣言の一覧が含まれており、例えば`late Config<int> _intConfig;`の場合、`node.fields.variables`には`_intConfig`というフィールド宣言が含まれます。

#### 2. `methodInvocations`からメソッド宣言一覧を取得

`methodInvocations`には、`MethodInvocation`型のノードの一覧が含まれています。
`methodInvocations`は、`compilationUnit`の中で`MethodInvocation`型のノードを探す処理で取得しているため、`compilationUnit`の中で呼び出されているメソッドの一覧が含まれています。

#### 3. `methodInvocation.realTarget`からメソッドの実行対象を取得

`methodInvocation.realTarget`には、メソッドの実行対象が含まれています。
例えば、`_intConfig.dispose()`の場合、`_intConfig`がメソッドの実行対象になります。

#### 4. メソッドの実行対象がフィールド名と一致し、メソッド名が`dispose`であることを確認

`methodInvocation.realTarget`が`SimpleIdentifier`型であることを確認し、`methodInvocation.methodName.name`が`dispose`であることを確認します。
`SimpleIdentifier`型は、プレフィックスや追加の構造を伴わない基本的な変数、関数、またはオブジェクトの名前を指します。
これにより、`_intConfig.dispose()`のような形で`dispose`メソッドが呼び出されているかを確認できます。

#### 5. フィールド宣言と同じフィールド名で`dispose`が一度も呼び出されていない場合は、Lintを表示

ここまでで、`_intConfig.dispose()`のような形で`dispose`メソッドが呼び出されているかを確認できました。
最後に、フィールド宣言と同じフィールド名で`dispose`が一度も呼び出されていない場合は、`FieldDeclaration`のASTノードに対してLintを表示します。

これで、下記のような形で警告が表示されることが確認できました！

![](/images/lint-detect-dispose-method-call.png)
*`Config`インスタンスの宣言から`dispose`メソッドの呼び出しがないことを検知した場合、Lintを表示*

## ルールのテスト

[Custom Lintとは](#custom-lintとは)で紹介したように、[custom_lint](https://pub.dev/packages/custom_lint)には組み込みのテストメカニズムが用意されています。

今回は、[riverpod_lint](https://pub.dev/packages/riverpod_lint)を参考に、`// expect_lint`を使用した組み込みテストメカニズムを使用して、ルールのテストを実装します。

### セットアップ

[riverpod_lint](https://pub.dev/packages/riverpod_lint)では、`packages/riverpod_lint_flutter_test`にテスト用のパッケージを作成しています。

同様に、`packages/flutterfire_lint_test`にテスト用のパッケージを作成し、`flutterfire_lint`パッケージや`Config`が定義されている`flutterfire_configurator`パッケージを依存関係に追加します。

:::message
custom_lintを使って作成されたLintルールを使用するためには、`custom_lint`パッケージと作成したLintルールのパッケージを依存関係に追加する必要があります。
:::

```bash
flutter create -t package packages/flutterfire_lint_test --project-name flutterfire_lint_test
```

```yaml:pubspec.yaml
name: flutterfire_lint_test
publish_to: none

environment:
  sdk: ^3.0.0

dependencies:
  flutter:
    sdk: flutter
  flutterfire_configurator:

dev_dependencies:
  custom_lint: ^0.5.7
  flutter_test:
    sdk: flutter
  flutterfire_lint:
    path: ../flutterfire_lint

dependency_overrides:
  flutterfire_configurator:
    path: ../flutterfire_configurator
```

### テストの実装

今回のテスト対象である、`dispose_config.dart`に対する`test/src/dispose_config.dart`ファイルを作成し、`// expect_lint`を使用した組み込みテストメカニズムを使用して、ルールのテストを実装します。

```dart:test/src/dispose_config.dart
import 'dart:async';

import 'package:flutter/material.dart';
import 'package:flutterfire_configurator/configurator.dart';

class MyWidget extends StatefulWidget {
  const MyWidget({super.key});

  @override
  State<MyWidget> createState() => _MyWidgetState();
}

class _MyWidgetState extends State<MyWidget> {
  late Config<int> _intConfig1;
  // expect_lint: dispose_config
  late Config<int> _intConfig2;

  final streamController = StreamController<int>();

  @override
  void initState() {
    _intConfig1 = Config(
      value: 1,
      subscription: streamController.stream.listen((event) {}),
    );
    _intConfig2 = Config(
      value: 2,
      subscription: streamController.stream.listen((event) {}),
    );
    debugPrint(_intConfig1.value.toString());
    debugPrint(_intConfig2.value.toString());
    super.initState();
  }

  @override
  Future<void> dispose() async {
    await _intConfig1.dispose();
    await streamController.close();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Container();
  }
}
```

上記のコードでは、`// expect_lint: dispose_config`を使用して、`dispose_config`ルールが適用されることをテストしています。
`_intConfig1`は`dispose`メソッドが呼び出されているため、`dispose_config`ルールに引っかかりませんが、`_intConfig2`は`dispose`メソッドが呼び出されていないため、`dispose_config`ルールに引っかかる想定です。

では、テストを実行してみましょう。

```bash
❯ cd packages/flutterfire_lint_test
❯ dart run custom_lint
Analyzing...                           0.0s

No issues found!
```

想定通り、`_intConfig2`に対して`dispose_config`ルールが適用されていることが確認できました！

## まとめ

今回は、[custom_lint](https://pub.dev/packages/custom_lint)を使って独自のLintルールを作成する方法を紹介しました。
Lintルールを作成することで、特定のコーディングパターンや慣習に合わせたルールを定義することが可能になります。

独自パッケージの開発でLintルールを作成する際は、ぜひこちらの記事を参考にしてみてください！

## 参考

https://pub.dev/packages/custom_lint
https://invertase.io/blog/announcing-dart-custom-lint
https://pub.dev/packages/riverpod_lint
https://note.com/jigjp_engineer/n/nd5c80713b39d
