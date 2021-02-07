---
title: "imagine-utlの使い方"
---

imagine-app では JSON Lines 形式のメタデータを標準入力から読み込むことができます。つまり、読み込みたいデータセットがある場合は、それらのデータを JSON Lines として標準出力へ出力するスクリプトを任意の言語で書く事ができます。
とはいえ、データセットごとに毎回スクリプトを書くのは少し面倒です。そこで、よくあるメタデータ読み込みについては、[imagine-utl](https://github.com/mpppk/imagine-utl)という別ツールを併用することで簡単に読み込めるようになっています。

## インストール

GitHub Releases からダウンロードし、パスが通っている場所に置いてください。

Mac か Linux の場合、以下のコマンドでもダウンロードできます。

```
# Linuxの場合、パスの"darwin"を"linux"に置き換える
$ curl -L https://github.com/mpppk/imagine-utl/releases/download/v0.1.0/imagine-utl_0.1.0_darwin_amd64.tar.gz \
  | tar xv
```

## ラベルごとにディレクトリに分類されたデータセットを読み込む

例えば以下のようなディレクトリ構造である、犬と猫の画像データセットがあるとします。

```
$ tree dataset
dataset/
├── dog
│   └── dog1.png
└── cat
    └── cat1.png
```

dog ディレクトリには犬画像、cat ディレクトリには猫画像が格納されているので、各画像が格納されているディレクトリ名をタグとして付与できると良さそうです。これは`imagine-utl load`コマンドで実現できます。

```shell
# ↓ --depthフラグにより、画像が格納された場所から上位1階層分のディレクトリ名がタグとして付与されたjsonを出力する
$ imagine-utl load --dir path/to/dataset --depth 1
{ "name": "dog1.png", "path": "dog/dog1.png", "boundingBoxes": [{ "TagName": "dog" }] }
{ "name": "cat.png", "path": "cat/cat.png", "boundingBoxes": [{ "TagName": "cat" }] }
...

$ imagine-utl load --dir path/to/dataset --depth 1 | imagine asset update --db dataset.imagine
# => 出力をimagineに渡すことで簡単に読み込める
```

複数のディレクトリ名を付与したい場合は--depth の値を変更します。例えば以下のようなディレクトリ構成で、dog/cat に加えて test/train のメタデータも付与したいとします。この場合、画像の格納場所から上位 2 階層分のディレクトリ名をタグとして付与するので、`--depth 2`を指定します。

```
$ tree dataset
dataset/
├── test
│   ├── cat
│   │   └── cat1.png
│   └── dog
│       └── dog1.png
└── train
    ├── cat
    │   └── cat1.png
    └── dog
        └── dog1.png
```

```shell
# ↓ --depthフラグにより、画像が格納された場所から上位2階層分のディレクトリ名がタグとして付与されたjsonを出力する
$ imagine-utl load --dir path/to/dataset --depth 2
{ "name": "dog1.png", "path": "test/dog/dog1.png", "boundingBoxes": [{ "TagName": "test" }, { "TagName": "dog" }] }
{ "name": "cat1.png", "path": "train/cat/cat1.png", "boundingBoxes": [{ "TagName": "train" }, { "TagName": "cat" }] }
...

$ imagine-utl load --dir path/to/dataset --depth 2 | imagine asset update --db dataset.imagine
# => 出力をimagineに渡すことで簡単に読み込める
```
