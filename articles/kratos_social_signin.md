---
title: ory kratosでソーシャルサインインを実装する
emoji: "🔐" 
type: "tech" 
topics: ["ory", "kratos", "authentication"] 
published: false
---  
## 本記事の概要
以前、ory kratos(セルフホスト版)を使用した、self-service flowの機能およびWebアプリケーションの実装サンプルを紹介しました。

@[card](https://zenn.dev/yoshinori_satoh/articles/kartos_usecase_overview)

@[card](https://zenn.dev/yoshinori_satoh/articles/kratos_browser_flow_example)

self-service flowは、kratos自身がIdP(Identity Provider)となり、ID/パスワードによるアカウント作成、ログイン、パスワードリセットなどの機能を提供します。

これに加えて、kratosではソーシャルプロバイダを利用したサインインもサポートしています。

kratosを使用したソーシャルサインインについて、以下を検証してみましたので、解説します。

* googleやgithubなどのソーシャルプロバイダを利用したサインイン
* ソーシャルプロバイダで不足しているユーザー属性(Traits)の補完
* self-service flowによって作成したアカウントおよび複数のソーシャルプロバイダ同士のアカウント紐付け

## サンプルコード
サンプルコードを以下リポジトリにて公開しています。

@[card](https://github.com/YoshinoriSatoh/kratos_example/tree/kratos_v1_1_0_social_signin)

## googleやgithubなどのソーシャルプロバイダを利用したサインイン


## self-service flowによって作成したアカウントおよび複数のソーシャルプロバイダ同士のアカウント紐付け


## ソーシャルプロバイダで不足しているユーザー属性(Traits)の補完


