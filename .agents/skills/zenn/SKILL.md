---
name: zenn
description: Zennの記事・本執筆時のMarkdown記法リファレンス。見出し、リスト、コードブロック、数式(KaTeX)、テーブル、脚注、独自記法(message, details)、コンテンツ埋め込み(YouTube, GitHub, X等)、mermaidダイアグラムの書き方を確認したいときに使用する。
---


# 記事（article）の管理

## ファイルの配置ルール

1 つの記事の内容は、1 つのmarkdownファイル（ `◯◯.md` ）で管理します。ファイルは `articles` という名前のディレクトリ内に含める必要があります。

```
.
└─ articles
   ├── example-article1.md
   └── example-article2.md
```

具体的な例として [Zenn のドキュメント用リポジトリ](https://github.com/zenn-dev/zenn-docs) を見ると分かりやすいかもしれません。

## 記事の作成

以下のコマンドによりmarkdownファイルを簡単に作成できます。

```
$ npx zenn new:article
```

`articles/ランダムなslug.md` というファイルが作成されます。slug（スラッグ）はその記事のユニークな ID のようなものです。詳しくは「 [Zenn の slug とは](https://zenn.dev/zenn/articles/what-is-slug) 」をご覧ください。

作成されたファイルの中身は次のようになっています。

```
---
title: "" # 記事のタイトル
emoji: "😸" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: [] # タグ。["markdown", "rust", "aws"]のように指定する
published: true # 公開設定（falseにすると下書き）
---
ここから本文を書く
```

👆 ファイルの上部には `---` に挟まれる形で記事の設定（Front Matter）が含まれています。ここに記事のタイトル（title）やトピックス（topics）などを **yaml 形式** で指定することになります。

コマンド実行時に記事の Front Matter をオプションで指定することもできます。

```
$ npx zenn new:article --slug 記事のスラッグ --title タイトル --type idea --emoji ✨
```

本文に画像を挿入するには

以下のいずれかの方法で本文に画像を挿入できます。

1. [Zennの画像のアップロードページ](https://zenn.dev/dashboard/uploader) （要ログイン）から画像をアップロードしてURLを貼り付ける
2. [GitHubリポジトリ内に画像を配置する](https://zenn.dev/zenn/articles/deploy-github-images)
3. Gyazoなどの外部サービスにアップロードした画像のURLを貼り付ける

## プレビューする

本文の執筆は、ブラウザでプレビューしながら確認できます。ブラウザでプレビューするためには次のコマンドを実行します。

```
$ npx zenn preview # プレビュー開始
```

![](https://static.zenn.studio/user-upload/jix99kz8oao4czphw2ogxm5sup9s)

👆 このように各記事をプレビューをしながら執筆できます。

## 記事を公開する

記事を zenn.dev 上で公開するには `published` オプションが `true` になっていることを確認したうえで、ファイルをコミットし、Zenn と連携されている GitHub リポジトリにプッシュします。  
Zenn と連携したリポジトリの登録ブランチにプッシュされると、同期（デプロイ）が開始されます。

なおコミットメッセージに `[ci skip]` もしくは `[skip ci]` が含まれていると Zenn でのデプロイがスキップされます。

## 日時を指定して記事を公開する（公開予約する）

公開時間を指定して記事を公開するには、Front Matterにて `published` を `true` にした上で、 `published_at` を指定します。 `published_at` のフォーマットは、 `YYYY-MM-DD` または `YYYY-MM-DD hh:mm` です。日付だけを指定した場合、時刻は `00:00` となります。

```
published: true # trueを指定する
published_at: 2050-06-12 09:03 # 未来の日時を指定する
```

この状態で、GitHubリポジトリへプッシュすると、zenn.dev上で記事が公開予約状態となり、公開予約時刻が過ぎると自動的に記事が公開されます。

## 過去の公開日時で記事を公開する

他のブログサービスなどからzenn.devに記事を移行する際に公開日時を維持したい場合、Front Matterにて `published_at` に過去の日時を指定することで、zenn.dev上での公開日時を指定することができます。 `published_at` のフォーマットは、 `YYYY-MM-DD` または `YYYY-MM-DD hh:mm` です。日付だけを指定した場合、時刻は `00:00` となります。

```
published: true # true/falseどちらでもOKです
published_at: 2010-01-01 08:00 # 過去の日時を指定する
```

## 記事の更新

記事の更新を行う場合も、markdownファイルを編集し、GitHub リポジトリへプッシュするだけで OK です。このとき slug が同一のものでないと別の記事として作成されてしまうので注意しましょう。

## 記事の削除

削除は [ダッシュボード](https://zenn.dev/dashboard) から行います。安全のため、 `articles` ディレクトリからmarkdownファイルを削除しても zenn.dev 上では削除はされません。

---

## CLI で本（book）を管理する

Zenn の本は複数のチャプターで構成されます。

## 本のディレクトリ構成

GitHub リポジトリで本のデータを管理する場合は、次のようなディレクトリ構成にします。

```
.
├─ articles
└─ books
   └── 本のスラッグ
       ├── config.yaml # 本の設定
       ├── cover.png　# カバー画像（JPEGかPNG）
       └── チャプターのスラッグ.md # 各チャプター
```

具体的には、以下のようになります。

```
# 具体的な例
books
└── my-awesome-book
    ├── config.yaml
    ├── cover.png
    ├── example1.md
    ├── example2.md
    └── example3.md
```

👆 `example1.md` や `example2.md` は各チャプターファイルの例です（のちほど詳しく説明します）。

これが 1 冊の本のファイル構成です。複数の本を作成するためには、同様の構成のディレクトリを複数 `books` 内に作ることになります。

## 本の各ファイルの役割

### 📄 config.yaml

本の設定ファイルです。以下のように記入してください。

```
title: "本のタイトル"
summary: "本の紹介文"
topics: ["markdown", "zenn", "react"] # トピック（5つまで）
published: true # falseだと下書き
price: 0 # 有料の場合200〜5000
chapters:
  - example1 # チャプター1
  - example2 # チャプター2
  - example3 # チャプター3
```

- **`title`**: 本のタイトルを入力します
- **`summary`**: 本の紹介文を入力します。これは有料の本であっても公開されます
- **`topics`**: トピック（タグ）を 5 つまで指定します
- **`published`**: 公開する場合は `true` にします
- **`price`**: たとえば本を 1000 円で販売するときは `price: 1000` と記載します（200〜5000 の間で 100 円単位で設定する必要があります）
- **`chapters`**: チャプターの並び順を配列で指定します（入れ子には未対応）。ここに指定されなかったチャプターは zenn.dev に同期されません
- **`toc_depth`**: 以下の説明をご覧ください

#### 🚀 目次の表示設定

2020/10/26 から、チャプターごとの目次を表示できるようになりました。デフォルトでは「h1」と「h2」見出しまで目次に表示されます。

`config.yaml` で `toc_depth: 0` と記載すると各チャプターの目次を非表示にできます。 `toc_depth: 1` とすると「h1」見出しだけが目次に表示されます。この値は `0` 〜 `3` の範囲で指定する必要があります。

※ この値は本全体での設定になります。チャプターごとに別の設定をすることはできません。

### 🖼️ カバー画像

本のカバー画像（表紙）は `cover.png` もしくは `cover.jpeg` というファイル名で配置します。  
推奨の画像サイズは **幅 500px・高さ 700px** です。他のサイズにした場合も最終的にこのサイズにリサイズされます。

### 📄 各チャプターのファイル（◯◯.md）

各チャプターのファイル名は「 `a-z0-9` 、ハイフン `-` 、アンダースコア `_` の 1〜50 字の組み合わせ」+ `.md` とします。この文字列は URL の一部となります。例えば `about.md` のチャプターの URL は `/ユーザー名/本のスラッグ/viewer/about` となります。

各チャプターのmarkdownファイルには Front Matter でタイトルを指定し、その下に本文を書いていきます。

```
---
title: "チャプターのタイトル"
---
ここからチャプター本文
```

本文に画像を挿入するには

以下のいずれかの方法で本文に画像を挿入できます。

1. [Zennの画像のアップロードページ](https://zenn.dev/dashboard/uploader) （要ログイン）から画像をアップロードしてURLを貼り付ける
2. [GitHubリポジトリ内に画像を配置する](https://zenn.dev/zenn/articles/deploy-github-images)
3. Gyazoなどの外部サービスにアップロードした画像のURLを貼り付ける

有料の本で一部のチャプターを無料公開する場合は `free: true` を指定してください。本の価格が ¥0（無料）のときはこの設定は無視されます。

```
---
title: "タイトル"
free: true
---
```

### 具体的な例

たとえば、次の 4 つのチャプターファイルを作ったとします。

- `abstract.md`
- `results.md`
- `introduction.md`
- `conclusion.md`

表示順は `config.yaml` で指定します。

```
# config.yaml
...省略...
chapters:
  - introduction
  - results
  - conclusion
```

👆 zenn.dev 上では `chapters` に配列で指定された順番にチャプターが並ぶことになります。この例の場合「introduction → results → conclusion」の順に並びます。 `abstract.md` は指定されていないため同期されません。

#### ファイル名（チャプター番号.slug.md）で並び順を管理することも

`config.yaml` でチャプターの並び順を指定すると、ディレクトリ内が煩雑になってしまうという場合には、ファイル名で並び順を制御することもできます。

詳しく読む

ファイル名を `チャプター番号.slug.md` という形にすると、その順番通りにチャプターが表示されます。

```
# 具体的な例
books
└── my-awesome-book
    ├── config.yaml
    ├── 1.intro.md
    ├── 2.foo.md
    └── 3.bar.md
```

この方法でデプロイを行う場合、 `config.yaml` の `chapters` は空にしておく必要があります。

デプロイ時に以下のようなファイルの読み方をするためです。

- `config.yaml` で `chapters` が指定されている場合 =\> 指定通りに `slug.md` を読みにいく
- `config.yaml` で `chapters` が空の場合 =\> `番号.slug.md` を読みにいく

[関連する GitHub Issue →](https://github.com/zenn-dev/zenn-editor/issues/45)

## 本の雛形をコマンドで作成する

本の構成は少し複雑ですが、下記のコマンドを使えば雛形を作成できます。

```
$ npx zenn new:book
# 本のslugを指定する場合は以下のようにします。
# npx zenn new:book --slug ここにスラッグ
```

あとは 1 つずつファイルを作成していけば OK です。

## 本と各チャプターをプレビューする

以下のコマンドにより、ブラウザ上でプレビューしながら執筆できます。

```
$ npx zenn preview
```

## 最大チャプター数

本のチャプターは1冊あたり最大100個まで作成できます。

## 本の更新

本の更新を行う場合も、ファイルを変更し、GitHub リポジトリへプッシュするだけで OK です。このとき slug（ディレクトリ名）が同一のものでないと別の本として作成されてしまうので注意しましょう。

## 本の削除

削除は [ダッシュボード](https://zenn.dev/dashboard) から行います。 `books` ディレクトリからファイルを削除しても zenn.dev 上では削除はされません。

# Zenn のmarkdown記法一覧

## 見出し

```
# 見出し1
## 見出し2
### 見出し3
#### 見出し4
```

## リスト

```
- Hello!
- Hola!
  - Bonjour!
  * Hi!
```

- Hello!
- Hola!
	- Bonjour!
	- Hi!

リストのアイテムには `*` もしくは `-` を使います。

### 番号付きリスト

```
1. First
2. Second
```

## テキストリンク

```
[アンカーテキスト](リンクのURL)
```

[アンカーテキスト](https://zenn.dev/)

Markdownエディタでは、テキストを範囲選択した状態でURLをペーストすることで選択範囲がリンクになります。（ [参照](https://info.zenn.dev/2024-02-08-editor-update) ）

## 画像

```
![](https://画像のURL)
```

![](https://static.zenn.studio/user-upload/gxnwu3br83nsbqs873uibiy6fd43)

### 画像の横幅を指定する

画像の表示が大きすぎる場合は、URL の後に半角スペースを空けて `=○○x` と記述すると、画像の幅を px 単位で指定できます。

```
![](https://画像のURL =250x)
```

![](https://static.zenn.studio/user-upload/gxnwu3br83nsbqs873uibiy6fd43)

### Altテキストを指定する

```
![Altテキスト](https://画像のURL)
```

![Altテキスト](https://static.zenn.studio/user-upload/gxnwu3br83nsbqs873uibiy6fd43)

### キャプションをつける

画像のすぐ下の行に `*` で挟んだテキストを配置すると、キャプションのような見た目で表示されます。

```
![](https://画像のURL)
*キャプション*
```

![](https://static.zenn.studio/user-upload/gxnwu3br83nsbqs873uibiy6fd43)  
*キャプション*

### 画像にリンクを貼る

以下のようにすることで画像に対してリンクを貼ることもできます。

```
[![](画像のURL)](リンクのURL)
```

## テーブル

```
| Head | Head | Head |
| ---- | ---- | ---- |
| Text | Text | Text |
| Text | Text | Text |
```

| Head | Head | Head |
| --- | --- | --- |
| Text | Text | Text |
| Text | Text | Text |

## コードブロック

コードは「\`\`\`」で挟むことでブロックとして挿入できます。以下のように言語を指定するとコードへ装飾（シンタックスハイライト）が適用されます。

> \`\`\`js
> 
> \`\`\`

```
const great = () => {
  console.log("Awesome");
};
```

シンタックスハイライトには Shiki を使用しています。  
[📄 対応言語の一覧 →](https://shiki.style/languages)

### ファイル名を表示する

`言語:ファイル名` と`:`区切りで記載することで、ファイル名がコードブロックの上部に表示されるようになります。

> \`\`\`js:ファイル名
> 
> \`\`\`

```
const great = () => {
  console.log("Awesome")
}
```

### diff のシンタックスハイライト

`diff` と言語のハイライトを同時に適用するには、以下のように `diff` と `言語名` を半角スペース区切りで指定します。

> \`\`\`diff js
> 
> \`\`\`

```
@@ -4,6 +4,5 @@
+    const foo = bar.baz([1, 2, 3]) + 1;
-    let foo = bar.baz([1, 2, 3]);
```

なお、 `diff` の使用時には、先頭に `+` 、 `-` 、 `>` 、 `<` 、 `半角スペース` のいずれが入っていない行はハイライトされません。

同時にファイル名を指定することも可能です。

> \`\`\`diff js:ファイル名
> 
> \`\`\`

```
@@ -4,6 +4,5 @@
+    const foo = bar.baz([1, 2, 3]) + 1;
-    let foo = bar.baz([1, 2, 3]);
```

## 数式

Zenn では **KaTeX** による数式表示に対応しています。  
KaTeXのバージョンは常に最新バージョンを使用します。

[📄 KaTeXがサポートする記法の一覧 →](https://katex.org/docs/support_table.html)

### 数式のブロックを挿入する

`$$` で記述を挟むことで、数式のブロックが挿入されます。たとえば

```
$$
e^{i\theta} = \cos\theta + i\sin\theta
$$
```

は以下のように表示されます。

e^{i\\theta} = \\cos\\theta + i\\sin\\theta

### インラインで数式を挿入する

`$a\ne0$` というように `$` ひとつで挟むことで、インラインで数式を含めることができます。たとえば a\\ne0 のようなイメージです。

## 引用

```
> 引用文
> 引用文
```

> 引用文  
> 引用文

## 脚注

脚注を指定するとページ下部にその内容が表示されます。

```
脚注の例[^1]です。インライン^[脚注の内容その2]で書くこともできます。

[^1]: 脚注の内容その1
```

脚注の例[^1] です。インライン[^2] で書くこともできます。

## 区切り線

```
-----
```

---

## インラインスタイル

```
*イタリック*
**太字**
~~打ち消し線~~
インラインで\`code\`を挿入する
```

*イタリック*  
**太字**  
~~打ち消し線~~  
インラインで `code` を挿入する

### インラインのコメント

自分用のメモをしたいときは HTML のコメント記法を使用できます。

```
<!-- TODO: ◯◯について追記する -->
```

この形式で書いたコメントは公開されたページ上では表示されません。ただし、複数行のコメントには対応していないのでご注意ください。

## Zenn 独自の記法

### メッセージ

```
:::message
メッセージをここに
:::
```

```
:::message alert
警告メッセージをここに
:::
```

### アコーディオン（トグル）

```
:::details タイトル
表示したい内容
:::
```

タイトル

表示したい内容

#### 要素をネストさせるには

外側の要素の開始/終了に `:` を追加します。

```
::::details タイトル
:::message
ネストされた要素
:::
::::
```

タイトル

## コンテンツの埋め込み

### リンクカード

```
# URLだけの行
https://zenn.dev/zenn/articles/markdown-guide
```

URL だけが貼り付けられた行があると、その部分がカードとして表示されます。

<iframe src="https://embed.zenn.studio/card#zenn-embedded__f330e027d67be" frameborder="0"></iframe>

また `@[card](URL)` という書き方でカード型のリンクを貼ることもできます。

アンダースコア \_ を含むURLが正しく認識されない場合

[markdownパーサの仕様](https://zenn.dev/catnose99/scraps/e94c8e789f846a) により、アンダースコア（ `_` ）を含むURLで、正しくURLが認識されないことがあります。

```
https://zenn.dev/__example__
```

> [https://zenn.dev/\_\_example\_\_](https://zenn.dev/__example__)

対処法

1. カード型のリンクとして表示したい場合は  
	`@[card](ここにURL)` という書き方をしてください
2. 単純にリンク化された URL を貼り付けたい場合は `<https://zenn.dev/__example__>` のような形で `<` と `>` で URL を囲むようにしてください

### X（Twitter）のポスト（ツイート）

```
# ポストのURLだけの行（前後に改行が必要です）
https://twitter.com/jack/status/20

# x.comドメインの場合
https://x.com/jack/status/20
```

以前は `@[tweet](ポストのURL)` の記法を採用していましたが、現在はポストのURLを貼り付けるだけで埋め込みが表示されます。

アンダースコア \_ を含む URL が正しく認識されない場合

[markdown パーサの仕様](https://zenn.dev/catnose99/scraps/e94c8e789f846a) により、URL の `/` の区切りの中に 2 つ以上アンダースコア（ `_` ）を含むと、自動リンクが途中で途切れてしまいます。

```
https://twitter.com/__example__/status/12345678910
```

> [https://twitter.com/\_\_example\_\_/status/12345678910](https://twitter.com/__example__/status/12345678910)

対処法

このような URL では `@[tweet](ポストのURL)` という書き方をしていただくようお願いします。

#### リプライ元のポストを非表示にする

リプライを埋め込んだ場合、デフォルトでリプライ元のポスト含まれて表示されます。 `ポストのURL?conversation=none` のようにクエリパラメータに `conversation=none` を指定すると、リプライ元のポストが含まれなくなります。

### YouTube

```
# YouTubeのURLだけの行（前後に改行が必要です）
https://www.youtube.com/watch?v=WRVsOCh907o
```

以前は `@[youtube](YouTubeの動画ID)` という記法を採用していましたが、現在は動画URLを貼り付けるだけで動画を埋め込むことができます。

### GitHub

GitHub上のファイルへのURLまたはパーマリンクだけの行を作成すると、その部分にGitHubの埋め込みが表示されます。

```
# GitHubのファイルURLまたはパーマリンクだけの行（前後に改行が必要です）
https://github.com/octocat/Hello-World/blob/master/README
```

上記のリンクは、以下のように表示されます。

<iframe src="https://embed.zenn.studio/github#zenn-embedded__9b38f877d743e" frameborder="0"></iframe>

#### 行の指定

GitHubと同じように、リンクの末尾に `#L00-L00` のような形で表示するファイルの開始行と終了行を指定することができます。

```
# コードの開始行と終了行を指定
https://github.com/octocat/Spoon-Knife/blob/main/README.md#L1-L3
```

上記のリンクは以下のように表示されます。

<iframe src="https://embed.zenn.studio/github#zenn-embedded__8cafb3c94f3f7" frameborder="0"></iframe>

また、開始行のみ指定することもできます。

```
# コードの開始行のみ指定
https://github.com/octocat/Spoon-Knife/blob/main/README.md#L3
```

上記のリンクは、以下のように開始行のみ埋め込まれて表示されます。

<iframe src="https://embed.zenn.studio/github#zenn-embedded__e645dde3a53b6" frameborder="0"></iframe>

#### テキストファイル以外は埋め込めません

埋め込めるファイルは、ソースコードなどのテキストファイルのみとなっています。  
もし画像などのファイルを指定した場合は、以下のような表示になります。

<iframe src="https://embed.zenn.studio/github#zenn-embedded__ca333b41b2c04" frameborder="0"></iframe>

### GitHub Gist

```
@[gist](GistのページURL)
```

GistのページURLを指定します。

特定のファイルだけ埋め込みたい場合は `@[gist](https://gist.github.com/foo/bar?file=example.json)` のようにクエリ文字列で`?file=ファイル名` という形で指定します。

### CodePen

```
@[codepen](ページのURL)
```

デフォルトの表示タブは `ページのURL?default-tab=html,css` のようにクエリを指定することで変更できます。

### SlideShare

```
@[slideshare](スライドのkey)
```

SlideShare の埋め込み iframe に含まれる`...embed_code/key/○○...`の `◯◯` の部分を入力します。

### SpeakerDeck

```
@[speakerdeck](スライドのID)

例:
@[speakerdeck](4f926da9cb4cd0001f00a1ff)
@[speakerdeck](4f926da9cb4cd0001f00a1ff?slide=24)
```

SpeakerDeck で取得した埋め込みコードに含まれる `data-id` の値を入力します。スライド番号も指定できます。

### Docswell

```
@[docswell](スライドのURL)
# もしくは
@[docswell](埋め込み用のURL)

例:
@[docswell](https://www.docswell.com/s/ku-suke/LK7J5V-hello-docswell)
@[docswell](https://www.docswell.com/s/ku-suke/LK7J5V-hello-docswell#p13)
@[docswell](https://www.docswell.com/s/ku-suke/LK7J5V-hello-docswell/13)
@[docswell](https://www.docswell.com/slide/LK7J5V/embed)
```

スライドのURL（ `https://www.docswell.com/s/{UserId}/{SlideId}-xxx-xxx` ）、もしくは埋め込み用のURL( `https://www.docswell.com/slide/{SlideId}/embed` )を入力します。スライド番号も指定できます。

### JSFiddle

```
@[jsfiddle](ページのURL)
```

[埋め込みオプション](https://docs.jsfiddle.net/embedding-fiddles) を指定する場合、iframe用の埋め込みURL（ `ページのURL + /embedded/{Tabs}/{Visual}/` ）を入力します。

### CodeSandbox

```
@[codesandbox](embed用のURL)
```

CodeSandbox では、各ページから埋め込み用の `<iframe>` を取得できます。この `<iframe>` に含まれる `src` の URL を括弧の中に入力します。

### StackBlitz

```
@[stackblitz](embed用のURL)
```

StackBlitz では、各ページから「Embed URL」を取得できます。取得した URL をそのまま括弧の中に入力します。

### Figma

```
@[figma](共有リンクのURL)
```

Figma では、デザインページで共有リンクを取得できます。取得したURLをそのまま括弧の中に入力します。

### オンラインエディターではモーダルから挿入可能

オンラインのエディターでは「+」ボタンを押すことで、外部コンテンツ埋め込み用のモーダルを表示できます。

![](https://static.zenn.studio/user-upload/t87wf3d7xgfv7cabv4a9lfr1t79q)

### その他の埋め込み可能なコンテンツ

オンラインエディターの埋め込みの選択肢としては表示されませんが、以下の埋め込み記法もサポートしています。

#### blueprintUE

```
@[blueprintue](ページのURL)

例：
@[blueprintue](https://blueprintue.com/render/0ovgynk-/)
```

[blueprintUE](https://blueprintue.com/) を埋め込むには、公開されているページのURLをそのまま括弧の中に入力します。

## ダイアグラム

[mermaid.js](https://mermaid-js.github.io/mermaid/#/) によるダイアグラム表示に対応しています。コードブロックの言語名を `mermaid` とすることで自動的にレンダリングされます。

```
\`\`\`mermaid
graph TB
    A[Hard edge] -->|Link text| B(Round edge)
    B --> C{Decision}
    C -->|One| D[Result one]
    C -->|Two| E[Result two]
\`\`\`
```

は以下のように表示されます。

<iframe src="https://embed.zenn.studio/mermaid#zenn-embedded__e6c74de6b4248" frameborder="0"></iframe>

他にもシーケンス図やクラス図が表示できます。文法は mermaid.js に従っていますので、どのように書けばよいかは [公式サイトの文法](https://mermaid-js.github.io/mermaid/#/flowchart) を参照してください。

### 制限事項

Zenn で mermaid.js 対応を行うにあたり、いくつか制限事項を設定させていただいています。制限事項は今後も様子を見て追加・廃止・値の変更など行う可能性があります。

#### クリックイベントの無効化

[Interaction機能](https://mermaid-js.github.io/mermaid/#/classDiagram?id=interaction) として図の要素にクリックイベントなどが設定できますが、セキュリティの観点でZennでは無効にさせていただきます。

#### ブロックあたりの文字数制限 - 2000文字以内

ブロックあたりの文字数を **2000** 文字に制限させていただいています。これを超えた場合、ダイアグラムが表示される代わりにエラーメッセージが表示されます。

#### ブロックあたりのChain数制限 - 10以下

フローチャートにおいて、ノードをひとまとまりで表現する記述として `&` が利用できます。以下のようなイメージです。

```
\`\`\`mermaid
graph LR
   a --> b & c--> d
\`\`\`
```

は以下のように表示されます。

<iframe src="https://embed.zenn.studio/mermaid#zenn-embedded__8d878355c74cd" frameborder="0"></iframe>

便利ですが、数が多くなるとノードの接続が多くなり、ブラウザ側での描画に負荷が生じる可能性があるため、 `&` の数を **10** に制限させていただきます。こちらも超えた場合はダイアグラムの代わりにエラーメッセージが表示されます。


---

脚注

[pik](https://zenn.dev/pik) [2020/10/29](#comment-5c9050ee9b527e0173a3)

