---
title: "Getting Started"
---

このページでは、有名な画像データセットである[fashion mnist](https://github.com/zalandoresearch/fashion-mnist#)を利用して、imagine-app の使い方を紹介します。

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

docker 版もあるので、代わりに以下のコマンドでもダウンロードできます。こちらの場合、python などをインストールする必要はありません。

```shell
$ docker run \
    -u (id -u):(id -g) \
    -v (pwd)/fashion_mnist/train:/train \
    -v (pwd)/fashion_mnist/test:/test \
    mpppk/fashion_mnist_downloader
```

さらに docker-compose 版もあるので、以下でもダウンロードできます

```shell
$ docker-compose up
```

これで画像の準備ができました。

## imagine-app で画像を表示する

では imagine-app を起動してみましょう。以下のコマンドを実行します。(imagine-app をあらかじめダウンロードして、パスが通る場所に置いておく必要があります。まだインストールしていない方は[インストール手順]()をご覧ください)

```
$ imagine --db fashion_mnist.imagine
```

初期起動時は、以下のように画像が表示されません。

![](https://storage.googleapis.com/zenn-user-upload/g1ub7fwg6ohg5id43gea11pmmq9n)

これはどこに画像が置かれているかを imagine-app が知らないためです。画像が置かれているパスを base path と呼びます。base path は、画面上部の歯車マークをクリックし、Change ボタンを押すことで変更できます。この時、`Load assets from base directory`にチェックを入れておいてください。
クリックするとディレクトリ選択画面になるので、ダウンロードした fashion mnist のディレクトリを選択してください。

![](https://storage.googleapis.com/zenn-user-upload/zcgmdewxp0a8egbs480rp8eke8kd)

これで 以下のように画像が表示されます。

![](https://storage.googleapis.com/zenn-user-upload/yawx7bytz92rec6ce8w2r7wmfeu0)

これで読み込み完了です。

:::details 自分で fashion mnist 用の imagine ファイルを作成する
このチュートリアルでは作成済みのメタデータを利用しましたが、実際にはまず GUI か CUI から作成する必要があります。ここでは CUI で作成する場合について簡単に紹介します。

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

$ imagine-utl load --dir path/to/fashion_mnist --depth 2 | imagine asset update --db fashion_mnist.imagine
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

Bag タグが付与されました。再度クリックすると削除することができます。

## バウンディングボックスを設定する(WIP)

:::message alert
この機能は不完全です。まだ利用しないでください。
:::

画像左上に表示されている青色の四角をドラッグすることでバウンディングボックスを設定できます。オレンジ色の円をクリックすると削除されます。

![](https://storage.googleapis.com/zenn-user-upload/rnkuuc8bpbvqxjvlfzraqwlyiwlr)

## imagine-app のタグを他プログラムから利用する

imagine-app は付与したタグをエクスポートすることができるので、外部のプログラムから簡単に利用することができます。ここでは pandas でアノテーションを読み込む例を紹介します。

imagine-app ではアノテーションのエクスポートは CLI から行います。一度 imagine を閉じてから、以下のコマンドを実行してください。

:::message
imagine コマンドを並列で実行することはできません。2 つ目以降のコマンドは、他のコマンドが終了するまでブロックされます。
:::

```
$ imagine \
    --db fashion_mnist.imagine \
    --format csv \
    asset list > fashion_mnist.csv
```

fashion_mnist.csv には 1 行に画像一枚のアノテーション結果が書かれています。

```:assets.csv
"id","path","tags"
"1","test/Bag/0.png","test"
"2","test/Bag/1.png","Bag,test"
...
```

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

## 外部のアノテーションを imagine-app へ反映させる

例として、機械学習モデルが判定に失敗した画像へタグを付与する例を考えましょう。
リポジトリには以下のようなモデル判定結果を格納した csv が`predict.csv`として含まれています。

```csv
id, path, label, predict
2, test/Bag/1.png, Bag, Bag
2, test/Bag/1.png, Bag, Coat
```

`import_metadata.py`を実行する事で、`predict.csv`の内容を imagine-app で読み込める形式で出力します。

```shell
$ inference.py
{}
```

この出力を`imagine asset update`へパイプすることで、結果を反映させることができます。

```shell
$ inference.py | imagine --db fashion_mnist.imagine asset update
```

imagine を再度開いて、付与したタグで検索してみます。

```shell
$ imagine --db fashion_mnist.db
```

_「StartWith:inference-failed」で検索_

どの画像で判定に失敗したかがわかりやすいですね！

これで fashion mnist を利用したチュートリアルは終了です。次は他のデータセットで試してみてください！また、このチュートリアルで出てきたコマンドは[CUI モードの使い方]()、画面は[GUI モードの使い方]()でより詳しく説明されてますので、あわせてご覧ください。
