# Zenn Contents

[Zenn](https://zenn.dev/) の記事と本を GitHub リポジトリ連携で管理する。

* [📘 How to use](https://zenn.dev/zenn/articles/zenn-cli-guide)
* [📘 Markdown guide](https://zenn.dev/zenn/articles/markdown-guide)

## ディレクトリ構成

| パス | 用途 |
| --- | --- |
| `articles/` | 記事。ここに置いた `.md` は連携先ブランチへの push でそのまま Zenn へデプロイされる |
| `books/` | 本。1 冊 1 ディレクトリで、`config.yaml` にチャプターの並びを書く |
| `images/` | 記事・本から参照する画像。**リポジトリ直下のここだけが対象**。最初の画像を置く時に作る |
| `templates/` | 記事の雛形。Zenn のデプロイ対象ではないので、ここに置いたものは公開されない |
| `dataset/` | `books/imagine-app-doc` のチュートリアルで使うサンプル画像 |

## デプロイの仕組み

**GitHub Actions は要らない。** Zenn 側の GitHub 連携が、ダッシュボードに登録したブランチへの
push を検知して自動でデプロイする。このリポジトリのデフォルトブランチは `master`。

記事の公開・非公開はフロントマターの `published` が決める。デプロイされること自体は
公開を意味しない。**新規記事は `published: false` で作り、公開すると決めた時だけ true にする。**

参考: [アカウントにGitHubリポジトリを連携してZennのコンテンツを管理する](https://zenn.dev/zenn/articles/connect-to-github)

## PR で下書きデプロイ

デプロイ対象ブランチ宛の PR を作成・更新すると、その内容を Zenn 上で下書きとして確認できる。
マージ前に実物のレンダリングを見られるので、レビューの承認ゲートとして使える。

有効化（Zenn ダッシュボード側の設定であり、このリポジトリだけでは完結しない）:

1. ダッシュボードの「GitHubリポジトリ設定」で対象リポジトリを選ぶ
2. 「デプロイ対象ブランチに対するPRの作成・更新で下書きデプロイを実行する」をオンにする
3. 2026-05-18 以前から連携している場合は、GitHub App「Zenn Connect」の権限更新を Accept する

制約:

- 同一リポジトリ内の PR のみ。fork からの PR は対象外
- デプロイされるのは PR の `articles/` と新規 `images/`
- **公開済み・予約公開済みの記事は更新されない**
- PR をクローズ・マージしても、デプロイ済みの下書きと画像は自動削除されない

参考: [PRで下書きデプロイができるようになりました](https://info.zenn.dev/2026-08-26-personal-pr-draft-deploy)

## 画像

- リポジトリ直下の `images/` に置く。中の構造は自由
- 対応する拡張子は `.png` `.jpg` `.jpeg` `.gif` `.webp` のみ
- 1 ファイル 3MB 以内
- 参照は `/images/` から始まる**絶対パス**。相対パスは効かない
- リポジトリから消すと Zenn 上からも消えるので、参照されている画像は消さない
- **対応拡張子以外のファイルを置くとデプロイ時にエラーになる。**
  `.gitkeep` のような空ディレクトリ用のファイルも置けないので、`images/` は最初の画像と一緒に作る

参考: [GitHubリポジトリ連携で画像をアップロードする方法](https://zenn.dev/zenn/articles/deploy-github-images)

## ローカルでの書き方

```sh
yarn install
yarn new:article   # articles/ に雛形を生成する
yarn preview       # http://localhost:8000 でプレビュー
```

`templates/` の雛形から書き始める場合は、`articles/<slug>.md` へコピーする。

- `templates/article-explainer.md` — SaaS やフレームワークの使い方と、その思想を解説する記事
- `templates/article-comparison.md` — ある観点で複数の SaaS やフレームワークを比較する記事
