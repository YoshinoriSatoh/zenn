---
title: kratos upgrade v0.10.1 to v0.11.1
emoji: "📦" 
type: "tech" 
topics: ["ory", "kratos", "authentication"] 
published: true
---  

## 本記事の概要
kratosを導入しているシステムで、kratos versionをv0.10.1からv0.11.1にアップグレードしたので、具体的な手順を記載します。

### 公式ドキュメント
https://www.ory.sh/docs/kratos/guides/upgrade

### 手順
1. [CHANGELOG.md](https://github.com/ory/kratos/blob/master/CHANGELOG.md)を確認してBreaking Changeを確認する
2. データバックアップ
3.  [Ory Kratos SDK](https://www.ory.sh/docs/kratos/sdk/overview)のアップデート
4. 新バージョンのkratosデプロイ
5. [`kratos migrate sql`](https://www.ory.sh/docs/kratos/cli/kratos-migrate-sql)の実行


### 1. Breaking changesの確認

[0.11.0-alpha.0.pre.2 (2022-11-28)](https://github.com/ory/kratos/blob/master/CHANGELOG.md#0110-alpha0pre2-2022-11-28)
```
This patch changes the behavior of the recovery flow. It introduces a new strategy for account recovery that sends out short "one-time passwords" (`code`) that a user can use to prove ownership of their account and recovery access to it. This PR also updates the default recovery strategy to `code`.

This patch invalidates recovery flows initiated using the Admin API. Please re-generate any admin-generated recovery flows and tokens.

This is a breaking change, as it removes the `courier.message_ttl` config key and replaces it with a counter `courier.message_retries`.

Closes [#402](https://github.com/ory/kratos/issues/402) Closes [#1598](https://github.com/ory/kratos/issues/1598)

SDK Method `getJsonSchema` was renamed to `getIdentitySchema`.
```

```
このパッチは復旧フローの動作を変更します。アカウント回復のための新しい戦略が導入され、ユーザがアカウントの所有権を証明するために使用できる短い「ワンタイムパスワード」(コード)が送信され、そのアカウントへの回復アクセスが可能になります。このPRはまた、デフォルトのリカバリ戦略をコードに更新します。

このパッチはAdmin APIを使用して開始されたリカバリフローを無効にします。管理者が生成したリカバリフローとトークンはすべて生成し直してください。

これは、courier.message_ttl 設定キーを削除し、courier.message_retries カウンタに置き換えるという、画期的な変更です。

クローズ #402 クローズ #1598

SDKメソッドgetJsonSchemaの名前がgetIdentitySchemaに変更されました。
```

アカウント復旧方法に`code`が追加され、それがデフォルトの挙動になります。

これ以前のアカウント復旧方法には `link`が使用されていたため、影響を出さないためにはRecovery flowでlinkを使用するように明示的に指定する必要があります。

また、アカウント復旧に関する記述しかないのですが、verification flowにも同様の設定があるので、おそらくこちらにも必要と思われます。

```yaml
selfservice:
  flows:
    recovery:
      use: link
    verification:
      use: link
```

もしもAdminAPIを使用したリカバリーフローがある場合は無効化されてしまうため、もしサービスで（管理画面等で）リカバリー操作が実装されている場合は、ユーザーに事前お知らせしておくなどのケアが必要になりそうです。

SDKでgetJsonSchemaを使用している場合は、getIdentitySchemaに変更する必要があります。s

[0.11.1 (2023-01-14)](https://github.com/ory/kratos/blob/master/CHANGELOG.md#0111-2023-01-14)

```
The `/admin/courier/messages` endpoint now uses `keysetpagination` instead.
```

```
/admin/courier/messagesエンドポイントでは、代わりにkeysetpaginationを使用するようになりました。
```

もし上記エンドポイントを使用していたら、ページネーションの方式が変わるのかな？

（使用していないのでそんなに深く調べてません）

### 2. データバックアップ
kratosのバージョンアップ後に、kratos DBのマイグレーションが必要になります。

念の為、データ損失に備えてデータバックアップを取得しておき、有事の際にすぐにリストアできる手順を確立しておくべきです。

### 3. [Ory Kratos SDK](https://www.ory.sh/docs/kratos/sdk/overview)のアップデート
Golangを使用してAPIサーバを構築しているため、[ory/kratos-client-go](https://github.com/ory/kratos-client-go)を使用しています。

以下のタグを参照して、kratosのバージョンと一致したSDKを使用するようにします。

https://github.com/ory/kratos-client-go/tags

これは手順3になってますが、実際にはBreaking Changeの確認後に行うことになると思います。

### 4. 新バージョンのkratosデプロイ
v0.11.1のkratosをデプロイします。

実際にバージョンアップした環境は、AWS ECSを使用していたため、kratos v0.11.1のimageをデプロイしました。

この時点では、kratos DBのマイグレーションがまだ完了していないため、kratosアクセス時に以下のようなエラーが出ます。

```json
{
  "audience": "application",
  "error": {
    "message": "migrations have not yet been fully applied"
  },
  "http_request": {
    "headers": {
      "accept": "application/json",
      "accept-encoding": "gzip",
      "referer": "http://localhost:4534/health/ready",
      "user-agent": "OpenAPI-Generator/1.0.0/go"
    },
    "host": "localhost:4534",
    "method": "GET",
    "path": "/admin/health/ready",
    "query": null,
    "remote": "127.0.0.1:58734",
    "scheme": "http"
  },
  "http_response": {
    "status_code": 503
  },
  "level": "error",
  "msg": "An error occurred while handling a request",
  "service_name": "Ory Kratos",
  "service_version": "v0.11.1",
  "time": "2024-01-30T14:49:31.859107901Z"
}
```

公式の手順では、先にkratosをデプロイするように記載されているため、このタイミングでダウンタイムが発生します。

このため、本番環境の更新の際は、メンテナンスを入れる必要がありそうです。

先にDBマイグレーションをしたらどうなるのかとも思ったものの、それはそれでうまく動かない場合もありそうなので、公式の手順に従うことにしました。

### 5. [`kratos migrate sql`](https://www.ory.sh/docs/kratos/cli/kratos-migrate-sql)の実行

kratos DBにアクセス可能な端末もしくはサーバーから、karatos migrate sqlを実行します。

```
kratos migrate sql $DB_KRATOS_DSN -c .config.yml
```

特にエラーが出なければ、マイグレーション完了です。

あとは一通りの動作確認を実施して、kratosのバージョンアップは完了です。

## まとめ
kratosのバージョンをv0.10.1からv0.11.1にアップグレードする手順を記載しました。

今回のバージョンアップでは、Breaking changeの影響はありませんでしたが、それでもDBやSDKのバージョンも追従して更新する必要があるので、更新時には確認事項がそれなりにあるなという印象です。

また、ダウンタイムが発生するので、しっかりメンテナンスを入れる必要もありそうです。

kratosは単体でマルチテナントに対応していないため、例えば顧客単位でテナントを分離したいような場合には、kratosインスタンスおよびDBを増やしていく実装が選択肢としてあります。

kratosインスタンスが大量にある場合、バージョン更新が結構大変になりそうですね。


