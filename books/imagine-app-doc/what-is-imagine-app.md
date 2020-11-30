---
title: "imagine-appとは"
---

[imagine-app](https://github.com/mpppk/imagine) は、画像やそのアノテーションを管理するためのツールです。類似ツールである[microsoft/VoTT](https://github.com/microsoft/VoTT)や[tzutalin/labelImg](https://github.com/tzutalin/labelImg)との違いとして、検索機能や外部プログラムとの連携に重点を置いています。

## 主な機能

### まあまあ速い画像検索

![](https://storage.googleapis.com/zenn-user-upload/bmfkvavqzyetd6ti48sl9a8icyr7)

特定の物体が写っている画像だけを表示するなど、付与したアノテーションを元に画像検索できます。10 万枚の画像検索が数秒でできるので、まあまあ速いと思います。頑張るともっと速くなるのですが、まあまあ速いので当分いいかなという気持ちです。

### CUI モード

![asset-update](https://storage.googleapis.com/zenn-user-upload/9tir14lgbpmpsizx1eiljbsjh7tn)

imagine-app では、GUI による画像管理だけでなく、CLI 操作も可能なため、他プログラムとの連携が簡単です。python の実行結果を CLI 経由で反映させたり、CI 上で imagine-app を動作させることを想定しています。詳細な使い方はこちら。

### 高速な画像読み込み

![](https://storage.googleapis.com/zenn-user-upload/hthqy4qh63x7b1gcvc5x59bngk3s)

画像サイズにもよりますが、1 万枚/sec 以上の読み込み速度を確認しています。 また、読み込みはバックグラウンドで実行されるため、どれだけ巨大なデータセットでも読み込み開始直後から作業を開始することができます。

## そんなに主でもない機能

### 単一バイナリで動作

![](https://storage.googleapis.com/zenn-user-upload/71ud1q2yzkf35944xb5mfaydq8hn)

imagine-app はシングルバイナリで動きます。手元でも CI 上でも導入が簡単です。

### 1 ファイルに全てのメタデータを保存

アノテーションなどのメタデータは単一ファイルへ書き出されるため、バックアップや共有が簡単です。

## How it works

imagine-app は Go と React.js で書かれた Web アプリです。画像データセットを static file server で配信し、Chrome で表示しています。内部アーキテクチャについてご興味がある方はこちらをご覧ください。

## TODO

まだ色々な機能が足りてません。例えば以下です。

- 複数のデータセットを管理するための workspace という概念がありますが、まだ default-workspace 以外は使えません。
- アノテーション付与時にバウンディングボックスの位置を決められますが、あまり意味のある値が記録されません※1。現時点ではラベルの付与のみに留めて、ボックスを細かく調整するのは待ってください。
  - ※1 正確には、画像が 500px \* 500px だったときのバウンディングボックス情報が付与されます。
