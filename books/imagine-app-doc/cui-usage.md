---
title: "CLIモードの使い方"
---

imagine-app の特徴の一つは、GUI だけでなく CUI でもほとんどの操作が可能な点です。
メタデータ操作の自動化や CI パイプラインとの統合に便利です。

## CLI の基本

imagine の CLI モードでは、ほとんどのコマンドで`--db`フラグで読み込む imagine ファイルを指定する必要があります。
指定したファイルが存在しない場合は新規に作成します。

```
$ imagine --db db.imagine
```

以降は`--db`フラグを省略して説明します。

## メタデータのエクスポート

`asset update`コマンドで、imagine ファイルで管理されている画像とそのメタデータを出力します。
他のプログラムからメタデータを読み込む必要がある場合に利用します。

デフォルトでは JSON Lines 形式で出力します。

```
$ imagine asset list
{"id":1,"name":"0","path":"test/Bag/0.png","boundingBoxes":[{"id":1,"tagID":1,"x":0,"y":0,"width":0,"height":0},{"id":2,"tagID":2,"x":0,"y":0,"width":0,"height":0}]}
{"id":2,"name":"1","path":"test/Bag/1.png","boundingBoxes":[{"id":0,"tagID":1,"x":0,"y":0,"width":0,"height":0},{"id":0,"tagID":1,"x":0,"y":0,"width":0,"height":0}]}
```

`--format`フラグで`csv`を指定すると、csv 形式で出力します。

```
$ imagine --db db.imagine --format csv
"id","path","tags"
"1","test/Bag/0.png","test"
"2","test/Bag/1.png","Bag,test"
```

## メタデータの更新

メタデータの更新は`asset update`コマンドで行います。JSON Lines 形式のメタデータ情報を渡すことで更新できます。JSON のスキーマや仕様については準備中です。今のところはテストコードを見て察してください。

```
# 何らかの方法で更新用のJSON Linesを出力。ここではpythonファイルを例にしているが、何でも良い。
$ cat update.py
{"id":1,"name":"0","path":"test/Bag/0.png","boundingBoxes":[{"id":1,"tagID":1,"x":0,"y":0,"width":0,"height":0},{"id":2,"tagID":2,"x":0,"y":0,"width":0,"height":0}]}
{"id":2,"name":"1","path":"test/Bag/1.png","boundingBoxes":[{"id":0,"tagID":1,"x":0,"y":0,"width":0,"height":0},{"id":0,"tagID":1,"x":0,"y":0,"width":0,"height":0}]}

# 更新用のJSON Linesを標準入力経由で渡すことで、更新される
$ imagine asset update < update.py
```
