---
title: "Getting Started"
---

このページでは、有名な画像データセットである[fashion mnist](https://github.com/zalandoresearch/fashion-mnist#)を利用して、imagine-app の使い方を紹介します。
imagine-app をまだインストールしていなければ、[手順]()を参考にインストールしてください。

:::details fashion mnist とは?
fashion mnist はバッグやコートなど 10 クラスのファッション関連画像で構成されているデータセットです。28\*28 の画像が train として 6 万枚、test として 1 万枚の合計 7 万枚含まれています。
:::

## 事前準備

- Chrome のインストール
- imagine-app のインストール

## サンプルリポジトリのダウンロード

fashion mnist を imagine-app で利用するチュートリアルを imagine-samples リポジトリに用意しています。以下のコマンドでダウンロードし、チュートリアル用のディレクトリに移動してください。

```shell
$ git clone https://github.com/mpppk/imagine-samples
$ cd imagine-samples/fashion_mnist
```

## データセットの準備

まずは fashion mnist データセットをダウンロードして、png として保存します。リポジトリ内の python スクリプトを実行してダウンロードしてください(python や keras を事前にインストールしておく必要があります)。なお、スクリプトは [Fashion-MNIST のデータを仕分けして PNG ファイルで保存](https://qiita.com/kaityo256/items/e34778f3b4712e0660c2) を参考にさせていただきました。

```shell
python download.py
```

:::details 代わりに docker or docker-compose でダウンロードする

docker 版もあるので、代わりに以下のコマンドでもダウンロードできます。こちらの場合、python などをインストールする必要はありません。

```shell
$ docker run \
    -u (id -u):(id -g) \
    -v (pwd)/dataset/train:/train \
    -v (pwd)/dataset/test:/test \
    mpppk/fashion_mnist_downloader
```

docker-compose 版もあります。

```shell
$ docker-compose up downloader
```

:::

これで画像の準備ができました。`dataset`ディレクトリ以下に fashion mnist の画像がダウンロードされているはずです。

```shell
$ tree -L 3 .
.
├── dataset
│   ├── test
│   │   ├── Bag
│   │   ├── Boot
│   │   ├── Coat
│   │   ├── Dress
│   │   ├── Pullover
│   │   ├── Sandal
│   │   ├── Shirt
│   │   ├── Sneaker
│   │   ├── Top
│   │   └── Trouser
│   └── train
│       ├── Bag
(略)
```

## imagine-app で画像を表示する

では imagine-app を起動してみましょう。以下のコマンドを実行します。(imagine-app をあらかじめダウンロードして、パスが通る場所に置いておく必要があります。まだインストールしていない方は[インストール手順]()をご覧ください)

```
$ imagine --db db.imagine --basepath $(PWD)/dataset
```

以下のように画像が表示されるはずです。左側はデータセットの画像一覧、右側はこのデータセットに存在するタグの一覧です。

![](https://storage.googleapis.com/zenn-user-upload/yawx7bytz92rec6ce8w2r7wmfeu0)

画像を選択すると中央に表示され、付与されているタグがハイライトされます。

![](https://storage.googleapis.com/zenn-user-upload/q0jg87cw8vcdx0eyhe1onr6gqn2u)

:::details 自分で fashion mnist 用の imagine ファイルを作成する
このチュートリアルでは作成済みの imagine-app 用 DB ファイル(db.imagine)を利用しましたが、実際にはまず GUI か CUI から作成する必要があります。ここでは CUI で作成する場合について簡単に紹介します。

imagine-app は`asset update`コマンドで標準入力からメタデータを読み込むことができます。以下のような、1 行に一つの asset に対応する json lines 形式のメタデータを与えてください。(json を改行すると正しく動作しません)

```jsonl
 { "name": "1", "path": "test/Coat/1.png", "boundingBoxes": [{ "TagName": "test" }, { "TagName": "Coat" }] }
 { "name": "2", "path": "test/Coat/2.png", "boundingBoxes": [{ "TagName": "test" }, { "TagName": "Coat" }] }
```

あとは、データセットに応じて上記の json を出力するスクリプトを任意の言語で書けば OK です。fashion mnist の場合、ファイルパスからタグを抽出するようなコードを書けば良いでしょう。

とはいえ、データセットごとに毎回スクリプトを書くのは少し面倒です。そこで、よくあるメタデータ読み込みについては、[imagine-utl](https://github.com/mpppk/imagine-utl)という別ツールを併用することで簡単に読み込めるようになっています。fashion mnist の場合、`imagine-utl load`の depth オプションを利用してディレクトリ名をタグに反映させることで、簡単に読み込むことができます。

```shell
# ↓ --depthフラグにより、画像が格納された場所から上位2階層分のディレクトリ名がタグとして付与されたjsonを出力する
$ imagine-utl load --dir path/to/fashion_mnist --depth 2
{ "name": "1", "path": "test/Coat/1.png", "boundingBoxes": [{ "TagName": "test" }, { "TagName": "Coat" }] }
{ "name": "2", "path": "test/Coat/2.png", "boundingBoxes": [{ "TagName": "test" }, { "TagName": "Coat" }] }
...

$ imagine-utl load --dir path/to/fashion_mnist --depth 2 | imagine asset update --db db.imagine
# => 出力をimagineに渡すことで簡単に読み込める
```

`imagine asset update`コマンドの詳細な利用方法については、[CLI モードの使い方](./cui-usage.md)を、imagine-utl は[imagine-utl の使い方]()をそれぞれ参照してください。

:::

## 画像を検索する

テストデータのコートだけを表示してみましょう。画面左上のフィルタボタンをクリックします。

![](https://storage.googleapis.com/zenn-user-upload/6lytwf6s98cgts2bstzcwkq95dud)
_Add→ ドロップダウンから Equals を選択 →「test」と入力 → 同様に「Coat」と入力 →Apply をクリック_

テストデータのコート画像だけが表示されました。

![](https://storage.googleapis.com/zenn-user-upload/5qmxyv7gtgx51sf0sw8t8gi8zl7l)

## imagine-app でタグ(アノテーション)を付与する

次にアノテーションを変更してみましょう。適当なコート画像に、あろうことかバッグ(Bag)タグを付与してみます。適当なコート画像をクリックした後、右側のタグ一覧画面から「Bag」をクリックしてください。

![](https://storage.googleapis.com/zenn-user-upload/b492esq2x69ypsw5xf9zcqhf9d23)

Bag タグが付与されたことで、ハイライトされました。再度クリックすると削除することができます。

## バウンディングボックスを設定する(WIP)

:::message alert
この機能は不完全です。まだ利用しないでください。
:::

画像左上に表示されている青色の四角をドラッグすることでバウンディングボックスを設定できます。オレンジ色の円をクリックすると削除されます。

![](https://storage.googleapis.com/zenn-user-upload/rnkuuc8bpbvqxjvlfzraqwlyiwlr)

## imagine-app のタグを他プログラムから利用する

imagine-app は付与したタグを json や csv としてエクスポートすることができるため、外部のプログラムから簡単に利用することができます。

imagine-app ではアノテーションのエクスポートは CLI から行います。一度 imagine を閉じてから、以下のコマンドを実行してください。

:::message
imagine コマンドを並列で実行することはできません。2 つ目以降のコマンドは、他のコマンドが終了するまでブロックされます。
:::

```
$ imagine \
    --db db.imagine \
    --format csv \
    asset list > fashion_mnist.csv
```

`fashion_mnist.csv` には 1 行に画像一枚のアノテーション結果が書き出されています。

```:assets.csv
"id","path","tags"
"1","test/Bag/0.png","test"
"2","test/Bag/1.png","Bag,test"
...
```

:::details 利用例: pandas でエクスポートしたファイルを利用する
上記の手順で書き出した csv を pandas で読み込む例を紹介します。
タグは`tags`カラムにカンマ区切りで表現されているので、pandas 側で subset(train/test)と label の 2 つのカラムに変換します。

```python
import pandas as pd

csv_path = 'path/to/fashion_mnist.csv'

def select_first_label(tags):
  for label in tags.split(','):
    if label != "train" and label != "test":
      return label

def select_subset(df, subset):
  subset_df = df[
      df["tags"].str.contains(subset, na=False)
  ].assign(subset=subset)

  return subset_df.assign(
      label=subset_df["tags"].apply(select_first_label)
  ).dropna()

df=pd.read_csv(csv_path)
train_df = select_subset(df, "train")
test_df = select_subset(df, "test")

print(test_df.head(2))
```

出力:

| id  | path           | tags     | subset | label |
| --- | -------------- | -------- | ------ | ----- |
| 2   | test/Bag/1.png | test,Bag | test   | Bag   |
| 3   | test/Bag/2.png | test,Bag | test   | Bag   |

これでアノテーションを DataFrame として読み込めるようになりました。  
例えば keras では`ImageDataGenerator.flow_from_dataframe`からこの DataFrame をそのまま利用することができます。詳しくは[リポジトリの jupyter notebook]()を参照してください。

```python
train_generator=datagen.flow_from_dataframe(
  dataframe=train_df,
  directory=image_dir_path,
  x_col='path',
  y_col='label',
  subset="training",
  batch_size=300,
  seed=42,
  shuffle=True,
  class_mode="categorical",
  target_size=(img_width,img_height)
)
```

:::

## 外部のアノテーションを imagine-app へ反映させる

先ほどとは逆に、外部ツールで作成したアノテーションを imagine-app へ取り込んでみましょう。
例として、機械学習モデルの判定結果から、判定に失敗した画像に新しくタグを付与し、imagine-app 上で確認したいとします。 リポジトリには、判定結果を imagine-app にインポートするためのファイルである、`converted_incorrect_results.jsonl`が含まれています。

```json
# converted_incorrect_results.jsonl
{"path": "test/Bag/1.png", "boundingBoxes": [{"tagName": "inference-failed:Shirt"}]}
{"path": "test/Bag/10.png", "boundingBoxes": [{"tagName": "inference-failed:Shirt"}]}
```

このファイルは 1 行ごとに「どの画像(path)へどんなタグ(tagName)を付与するか」を記載しています。例えば、二行目の内容は、「`test/Bag/1.png`の画像へ、新しく`inference-failed:Shirt`というタグを付与する」という意味です。 つまり、本来は Bag の画像であるのに、間違えて Shirt と判定されてしまったことを表しています。
(ファイルフォーマットの詳細については、[asset update コマンドの使い方]()をご覧ください)

:::details インポート用の jsonl ファイル作成手順

リポジトリには以下のような、あるモデルによる判定結果のうち、間違った判定だけを格納した csv が`incorrect_predicts.csv`として含まれています。

```csv
Bag,Boot,Coat,Dress,Pullover,Sandal,Shirt,Sneaker,Top,Trouser,predict,actual,path
0.09600647,0.1303275,0.080824696,0.117921434,0.06790708,0.075323775,0.15818310000000002,0.07472554599999999,0.11275013,0.08603032,Shirt,Bag,test/Bag/1.png
0.09600647,0.1303275,0.080824696,0.117921434,0.06790708,0.075323775,0.15818310000000002,0.07472554599999999,0.11275013,0.08603032,Shirt,Bag,test/Bag/10.png
```

最初の 10 列は、機械学習モデルにより算出した各クラスである確率です。predict 列はモデルがどのクラスだと判定したか(確率が最大のクラス)、actual は実際のクラスです。この csv には誤判定結果だけが含まれているので、predict と actual は必ず異なるラベルになっています。

この結果を`imagine-app`で読み込める形式に変換したものが`converted_incorrect_results.jsonl`です。この場合、1 行ごとに predict と actual 列を参照し、imagine-app 用の json フォーマットへ変換していくことになります。具体的なコードはリポジトリの`convert_incorrect_predicts.py`をご覧ください)

:::

このファイルの内容を標準入力経由で imagine-app の update サブコマンドへ渡すことで、アノテーションのインポートを行うことができます。

```shell
$ imagine --db db.imagine asset update < converted_incorrect_results.jsonl
```

imagine を再度開くと、`inference-failed:XXX`というタグが、右側のタグ一覧に追加されているはずです。(新しいタグはリストの一番下に追加されるので、見当たらない場合は下にスクロールしてみてください。)

```shell
$ imagine --db fashion_mnist.db
```

誤判定の傾向を掴むために、付与したタグで検索してみましょう。例えば、「本来はコートなのに、異なるクラスとして判定されてしまった画像の一覧」を表示してみます。
誤判定を表すために付与したタグは全て「inference-failed」から始まるので、「Start With」クエリで指定することで、誤判定された画像だけに絞り込むことができます。

![](https://storage.googleapis.com/zenn-user-upload/m6f1stvyu08ncb8lxdahn9jxobs9)
_「inference-failed」で始まるタグと「Coat」タグを両方持つ画像だけを表示する検索クエリ_

![](https://storage.googleapis.com/zenn-user-upload/wzi5gtt92rnv9lfon331hplb3fmc)
_検索結果。実際はコートなのですが、プルオーバーと誤判定されているようです_

このように、外部ツールのアノテーション結果を imagine-app に取り込むことで、アノテーションに基づいた検索を行うことができます。

これで fashion mnist を利用したチュートリアルは終了です。次はあなたのデータセットで試してみてください！
また、このチュートリアルで出てきたコマンドは[CUI モードの使い方]()、画面は[GUI モードの使い方]()でより詳しく説明されていますので、あわせてご覧ください。
