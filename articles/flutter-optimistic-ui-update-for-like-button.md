---
title: "Riverpod と Flutter Hooks で Optimistic Update を実現する"
emoji: "❤️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "dart", "riverpod", "hooks"]
published: true
publication_name: "altiveinc"
---

## はじめに

こんにちは！Altive株式会社のFlutterアプリ開発者の小林遼太（[@naipaka](https://twitter.com/naipakapaka)）です🦙

この記事では、[riverpod](https://pub.dev/packages/riverpod) と [hooks_riverpod](https://pub.dev/packages/hooks_riverpod) を使って、いいねボタンを例に Optimistic Update （楽観的更新）を実現する方法を紹介します。

## Optimistic Update （楽観的更新）とは

「Optimistic UI Update」（楽観的なUI更新）はユーザーが操作を行った際に、その操作が成功することを前提に、即座にUIを更新する手法です。

「楽観的」という表現は、ユーザーが行った操作が成功するだろうという前提（楽観的な見通し）に基づいて、UIを先行して更新するという意味合いを持っています。

例えば、いいねボタンを押した際に、即座にいいねの状態を更新して、その後にAPIリクエストを送信することで、ユーザーの操作に対する応答が早くなり、UXが向上します。

## Optimistic Update の実現

Flutter のバージョンは `3.19.1` を使用します。
各パッケージのバージョンは下記のとおりです。

```yaml
dependencies:
  flutter_hooks: ^0.20.5
  hooks_riverpod: ^2.4.10
  riverpod_annotation: ^2.3.4

dev_dependencies:
  build_runner: ^2.4.8
  riverpod_generator: ^2.3.11
```

### 元となる実装

それでは、まずいいねボタンの Optimistic Update を実現するための元となる実装についてです。

まずは、API からいいねの情報を取得し提供するProviderを作成します。
通常は投稿情報などと一緒に取得することが多いと思いますが、今回は簡略化のためにいいねしたかどうかの情報のみを取得することにしています。

```dart
@riverpod
Future<bool> isLiked(IsLikedRef ref)async {
  // GET などの API リクエストでいいねの状態を取得する。
  // ...
}
```

次に、いいねボタンです。いいねの状態に応じていいねボタンのアイコンの表示が切り替わります。

```dart
class LikeOnlyPage extends ConsumerWidget {
  const LikeOnlyPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final asyncIsLiked = ref.watch(isLikedProvider);
    return Scaffold(
      body: Center(
        child: asyncIsLiked.when(
          loading: () => const CircularProgressIndicator(),
          error: (error, stackTrace) {
            return const Text('Error');
          },
          data: (isLiked) {
            return IconButton(
              icon: Icon(
                isLiked ? Icons.favorite : Icons.favorite_border,
              ),
              onPressed: () async {
                // POST などの API リクエストを送信する。
                // ...

                // いいねの状態を再取得する。
                ref.invalidate(isLikedProvider);
              },
            );
          },
        ),
      ),
    );
  }
}
```

この実装では、いいねボタンを押した際に、APIリクエストを送信し、その後にいいねの状態を再取得しています。
この形だと、リクエストが完了するまでUIが変わらないので、ユーザーの操作に対する応答が遅くなってしまいよくないですね。

### Optimistic Update 対応

上記の実装を元に、Optimistic Update を実現します。

まずは、`IconButton` を `HookBuilder` でラップし、`useState` を使って`isLikedProvider`とは別でいいねの状態を管理します。
`HookBuilder` を使用せず、 `HookConsumerWidget` を継承させる形でも問題ないです。

```dart
          ...
          data: (isLiked) {
            return HookBuilder(
              builder: (context) {
                final isLikedState = useState(isLiked);
                return IconButton(
                  icon: Icon(
                    isLikedState.value ? Icons.favorite : Icons.favorite_border,
                  ),
                  ...
```

次に、`onPressed` で`isLikedState`を更新してから、APIリクエストを送信する形に変更します。

また、APIリクエストの送信後に `ref.refresh` で`isLikedProvider`を再取得し、サーバー側のデータと同期します。
`useState` によって管理させる状態はウィジェットのビルドが再実行されてもその値が保持されるため、 `ref.invalidate` ではなく `ref.refresh` を使って明示的に状態を更新する必要があります。

```dart
                  onPressed: () async {
                    // いいねの状態を即座に更新する。
                    isLikedState.value = !isLikedState.value;

                    // POST などの API リクエストを送信する。
                    // ...

                    // いいねの状態を再取得する。
                    isLikedState.value = await ref.refresh(isLikedProvider.future);
                  },
```

エラーが発生した場合には `isLikedState` の状態を元に戻すようにします。

```dart
                  onPressed: () async {
                    // いいねの状態を即座に更新する。
                    isLikedState.value = !isLikedState.value;
                    
                    try {
                      // POST などの API リクエストを送信する。
                      // ...

                      // いいねの状態を再取得する。
                      isLikedState.value = await ref.refresh(isLikedProvider.future);
                    } catch (e) {
                      // エラーが発生した場合には、いいねの状態を元に戻す。
                      isLikedState.value = !isLikedState.value;
                    }
                  },
```

これで、いいねボタンの Optimistic Update を実現することができました！

:::details 完成形
```dart
class LikeOnlyPage extends ConsumerWidget {
  const LikeOnlyPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final asyncIsLiked = ref.watch(isLikedProvider);
    return Scaffold(
      body: Center(
        child: asyncIsLiked.when(
          loading: () => const CircularProgressIndicator(),
          error: (error, stackTrace) {
            return const Text('Error');
          },
          data: (isLiked) {
            return HookBuilder(
              builder: (context) {
                final isLikedState = useState(isLiked);
                return IconButton(
                  icon: Icon(
                    isLikedState.value ? Icons.favorite : Icons.favorite_border,
                  ),
                  onPressed: () async {
                    // いいねの状態を即座に更新する。
                    isLikedState.value = !isLikedState.value;

                    try {
                      // POST などの API リクエストを送信する。
                      // ...

                      // いいねの状態を再取得する。
                      isLikedState.value = await ref.refresh(isLikedProvider.future);
                    } catch (e) {
                      // エラーが発生した場合には、いいねの状態を元に戻す。
                      isLikedState.value = !isLikedState.value;
                    }
                  },
                );
              },
            );
          },
        ),
      ),
    );
  }
}
```
:::


## さいごに

今回は、[riverpod](https://pub.dev/packages/riverpod) と [hooks_riverpod](https://pub.dev/packages/hooks_riverpod) を使って、いいねボタンの Optimistic Update を実現する方法を紹介しました。

いいねボタンの他にも、スイッチやチェックボックスなど、ユーザーの操作に対する即座のフィードバックが求められる場面で活用できますね。

ぜひ、参考にしていただければ幸いです！
