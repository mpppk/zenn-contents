---
title: "内部アーキテクチャ(WIP)"
---

imagine-app は、 UI 実装に React(Next.js)、DB やファイルへのアクセスに Go を利用しています。
また、画面表示には Chrome を利用しているため、ユーザが予め Chrome をインストールしている必要があります。

## lorca

## lorca-fsa によるサーバ・クライアント間通信

imagine-app では、サーバ・クライアント間で Flux Standard Action(fsa)を用いて通信を行う [lorca-fsa](https://github.com/mpppk/lorca-fsa) というライブラリを利用しています。これにより、フロントエンドで発行した fsa をサーバ側でハンドリングし、fsa として結果をまたクライアント側へ通知する事ができます。
