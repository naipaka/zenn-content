# Zenn Content リポジトリについて

このリポジトリは、[Zenn](https://zenn.dev/)に投稿するための技術記事や技術書のコンテンツを管理しています。

## リポジトリ構成

```
articles/        - 技術記事のMarkdownファイル
books/           - 技術書のコンテンツ
images/          - 記事で使用する画像ファイル
```

## 記事の書き方

### 新しい記事の作成

Zenn CLIを使って新しい記事を作成します：

```bash
npx zenn new:article
```

### 記事のフロントマター（メタデータ）

各記事のMarkdownファイル冒頭には以下のようなフロントマターを設定します：

```md
---
title: "記事のタイトル"
emoji: "🔍"  # アイコンとして使用される絵文字
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["flutter", "dart", "riverpod"]  # タグ（5つまで）
published: true  # 公開設定（falseの場合は下書き）
published_at: 2023-12-17  # 公開日（省略可）
publication_name: "altiveinc"  # 所属組織のスラッグ（省略可）
---
```

### 画像の追加

`images/` ディレクトリに画像ファイルを配置し、記事内で以下のように参照します：

```md
![代替テキスト](/images/example-image.png)
```

### プレビュー

記事のプレビューは以下のコマンドで確認できます：

```bash
npx zenn preview
```

## 記事の執筆ポリシー

このリポジトリでは、主に以下のようなトピックに関する技術記事を執筆しています：

1. Flutter/Dartに関する技術情報
2. モバイルアプリ開発のベストプラクティス
3. その他、Web開発やクラウドサービスに関する知見

記事は読者にとって有用な情報を含み、具体的なコード例と説明を組み合わせて提供するようにしてください。

## 公開前のチェックリスト

- [ ] タイトルは明確で内容を適切に表現しているか
- [ ] 記事構成は論理的で流れがスムーズか
- [ ] コードサンプルは正確で動作確認済みか
- [ ] 画像は適切に配置され、内容の理解を助けているか
- [ ] 誤字脱字や文法的な間違いがないか
- [ ] フロントマターの設定は正しいか

## 参考リンク

- [Zenn CLI の使い方](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [Markdown記法の一覧](https://zenn.dev/zenn/articles/markdown-guide)