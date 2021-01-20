---
title: "便利な使い方"
---

便利な使い方を紹介します。

## 学習データとテストデータの分割

train や test などのラベルを付与しておく事で、同一ディレクトリで学習データとテストデータを管理する事ができます。

![](https://storage.googleapis.com/zenn-user-upload/20rguudjcuvtnkjvrefuc5wrz5u3)

## 特定の画像を除外する

現実のデータセットでは、学習データとして利用すべきでない画像が含まれている事があります。
例えば重複画像や、対象物が鮮明に写っていない画像などです。
imagine-app でのアノテーション時に、これらの画像へ"ignore"や"invalid"などのタグを付与しておく事で、pandas での読み込み時にそれらの画像を無視する事ができます。

![](https://storage.googleapis.com/zenn-user-upload/eqzv1tgalwfvuld4nk15uc8d9uq7)

## NoTags 検索を利用したアノテーション作業の効率化

imagine-app の検索クエリの一つに、NoTags があります。これは画像のうち一つもタグが付与されていない画像だけを表示するものです。1 万枚のデータセットのうち 2000 枚タグを付与した状態で次の日に 2001 枚目を探すのは大変ですが、NoTags を利用することで付与対象の画像だけを表示できます。

![](https://storage.googleapis.com/zenn-user-upload/duzsyj67pv2kmjxzrrwg6jb6alz7)

## 推論結果タグを利用したアノテーション作業の効率化
