---
title: ID管理 & 認証APIの ory kratos(self-hosting)の紹介
emoji: "📦" 
type: "tech" 
topics: ["ory", "kratos", "authentication"] 
published: false
---  

## 本記事の概要
Webアプリに必要不可欠なIdentity管理(ユーザー管理)および認証機能の導入について、どのような選択肢があるのか、その中でもory kratos(self-hosting)選択した背景、理由について記載します。

ory kratosの機能や使用する際のイメージ、設計Tipsと簡単なサンプル実装を紹介します。

## 認証実装手段の選択肢
認証機能はWebアプリの必須要件と言って良いかと思います。

認証機能の実現方法として、ざっと以下のような選択肢があるかと思います。

* 各種フレームワーク内蔵の認証機能を利用する
* IDaaS（クラウドサービス）を利用する
* セルフホスト型のOSSを利用する

一応、簡単に自前実装するという手段もないこともないですが、セキュリティリスクが高いと思うので、選択肢からは外します。

それぞれについて、ざっとメリデメを見てみましょう。

### フレームワーク内蔵の認証機能を利用する
最もポピュラーな選択肢ではないでしょうか。

フレームワークに含まれているため、キャッチアップし易く、メジャーなフレームワークであればドキュメントも充実しているため、トラブルシューティング等もし易いでしょう。

反面、Identityデータがフレームワークと同じDBに保管され、認証APIもフレームワークに依存します。

フレームワーク単体でシステムを稼働させており、負荷も大きくない場合は、特に問題とはなりません。

しかし、何らかの理由でシステム構成に変更を加えたくなった場合や、負荷へ対応することを考えると、柔軟性が低いと言えます。

例えば、新規で構築したAPIから既存の認証APIを利用したい場合、既存のフレームワークの認証APIを利用することになります。

勿論できないわけではないですが、ちょっとアーキテクチャが歪な感じがします。

また、高負荷への対応を考えると、フレームワークAPIのスケールアップ/スケールアウトが必要となり、認証部分だけの性能増強が難しいと思います。

### IDaaS（クラウドサービス）を利用する
Auth0やAmazon Cognitoのような、IDaaSを利用する選択肢もあります。

クラウドサービスなので、インフラを構築する必要がありません。

Auth0なんかは、利用についてもドキュメントが充実しており、プランによってはサポートも受けられるため、導入・運用が非常に簡単というサービスもあるでしょう。

クライアントから認証APIのエンドポイントを呼び出すだけで利用できるため、利用の敷居も低いです。

(Cognitoはややキャッチアップが面倒だった記憶があります。)

デメリットとしては、サービスによってはそれ相応のコストがかかる点です。(Auth0など)

料金体系はサービスによっても変わりますが、ユーザーが一定数を超えると、グレードが上のプランへ移行する必要があったり、SSO等の特定の機能もある程度以上のプランでないと利用できない場合があります。

実際に利用したことがないため、具体的な金額感はわかりませんが、後から大きなコストになりそうな気がして、個人的には導入に足踏みしてしまいます。

また、データがクラウドサービスに保管されるため、セキュリティ要件を満たせない場合があるかもしれません。

### セルフホスト型のOSSを利用する
認証APIサーバーを構築して利用する方法です。

自分たちでインフラ構築および運用が必要となりますが、データをクラウドに出すことなく、インフラのネットワーク内に格納できます。

バックエンドAPIとは別に認証APIを構築しているため、前述したフレームワーク依存問題を解決できるため、API新規追加等のシステム構成変更や、高負荷への対応がしやすいです。

OSSを利用するため、IDaaSのようなコストもかかることなく、インフラ費用としてはサーバー稼働費用のみとなります。

コスト面、データ保管場所、システム構成やスケーリングの観点から、セルフホスト型のOSSを採用したいケースは多いのかなと思います。　

## 複数チームによるAPI開発における認証の課題と解決手段としてのkratos
今後新規のWebアプリ開発する際に、認証をビジネスロジックのAPIから切り離した設計にしたいと考えていました。

関わっているプロジェクトのいくつかは、複数のチームがそれぞれAPIを開発するスタイルで進められているのですが、この場合に認証をどのチームが担当するのか、どのAPIで担当するべきなのかなど、認証について考えなければならないことがたくさん出てきます。

このように書くと、いわゆるマイクロサービスしている雰囲気がありますが、正直なところちゃんとマイクロサービスできているわけではありませんでした。

とはいえシステム全体の根幹に関わる認証については、部分的だとしてもなるべくちゃんとマイクロサービスな構成にしたいと考えていました。

複数あるAPIへどのようにルーティングするのか、どこで認証ガードさせるべきかを考えた結果、認証APIを独立させた上で、APIゲートウェイパターンで認証ガードさせる構成が良いと考えました。

### ory kratos導入背景
上記の構成を念頭に置いた際に、何かいいOSSがないものかと探していて、1年ちょっと前くらいに見つけたのが ory kratos でした。

kratosはIdentity管理および認証APIを提供するOSSです。（クラウドサービス版もありますが、ここではセルフホスト版について記載します。）

認証に関する一通りの機能が備わっており、MFAやソーシャルサインインのよう機能も備わっています。

当時はまだv0.10くらいのバージョンでしたが、現在はv1.0.0がリリースされています。

（ここ半年くらい新しいバージョンがリリースされてませんが、[masterブランチには追加機能や不具合修正が多くコミットされています。](https://github.com/ory/kratos/blob/master/CHANGELOG.md))

認証に関する機能的なアップデートも今後入っていくと考えると、kartosを導入しておく恩恵が期待できます。

また、Passkeyのような新しい機能についても、現時点はセルフホスト版では未実装のようですが、今後実装されることも個人的に期待しています。

([Passkeyはクラウド版には搭載されているようです](https://www.ory.sh/docs/kratos/passwordless/passkeys))

### kratosの機能
kratosは、ID管理および認証に必要な一通りのAPI/機能を提供します。

* ユーザー登録、ログイン、メールアドレス等の検証、アカウント復旧、ユーザープロファイル設定
* 上記の各種機能実行時のトリガーによるフック処理を記述可能
* 本人検証やアカウント復旧/検証時のメール送信 (テンプレートによるカスタムも可能)
* MFA
* ソーシャルサインイン
* アドミンによるユーザー管理

フレームワークに内蔵されている認証機能と同等以上の機能を持っていると思います。

### kratosのセキュリティ
kratosには、ユーザー自身によるユーザー登録やログイン、アカウント復旧といった、[SelfService flow](https://www.ory.sh/docs/kratos/self-service)と呼ばれる実装がなされており、この仕様がNISTやIFTF、Microsoft Research、Google Research、Trou Huntによって確立されたベストプラクティスに基づいているとのことです。

SelfService flowに従うことで、各種攻撃やCSRFに対するセキュリティが確保されます。

その他、各種機能の実装やパスワードポリシーに関して[NISTのデジタルIDガイドラインに準拠しています。](https://www.ory.sh/docs/kratos/concepts/security))

### kratosのスケーリング
kartosは、DBにのみ状態を持っているため、[コンテナの水平スケーリングによってスケールアウトが可能です。](https://www.ory.sh/docs/self-hosted/operations/scalability)

DBの負荷については気にする必要はありますが、kratosのDBのみを増強するなど、認証部分のみをスケールアウト/スケールアップすることも可能です。

DBは、MySQL、PostgreSQL、CockroachDBをサポートしています。

CockroachDBについては詳しくないですが、もしCockroachDBを導入できるのであれば、DBもスケーリングできるため、認証部分を細かくスケーリングできるようになりそうです。

使ってみた感触では、kratosはGolangで実装されているため、起動が早いためデプロイ等の際に非常に使いやすかったです。

### IAPの構成要素としてのkratos
oryは、kratosの他に幾つかのOSSを開発しており、その中の一つにory oathkeeperというプロキシがあります。

正確には、Identity & Access Proxy (IAP)というもので、リクエストが認証されているかどうかを検査し、認証されていれば後段のAPIへリクエストを転送するような機能を持っています。

oathkeeper自体がIAPとして振る舞うこともできますし、Traefikのような専用のプロキシ(エッジルーター)からoathkeeperを呼び出して、IAPを実装することもでき、kratosと相性も良いです。

## kratosユースケース概要
kratosを使用したユーザー登録からログイン、セッションの取得、APIへの受け渡しおよびAPI側でのセッションの取り扱いについて、ユースケース概要を描いてみました。

![](https://github.com/YoshinoriSatoh/zenn/blob/master/images/kartos_usecase_overview/kratos_usecase_overview.png?raw=true)
*kratosユースケース概要*

左側の枠がWeb/Native App、真ん中にkratosとその右側にkratos DB、下側にビジネスロジックAPIとその右側にAPI用のDBを記載しています。

kratosでは、ユーザー自身によるユーザー登録、ログイン、アカウント復旧といった、SelfService flowと呼ばれる実装がなされており、詳細は割愛していますが、図中に各SelfService flowと関連する流れとDB内のテーブルを記載しています。

kartosは、Webアプリ(ブラウザ)およびネイティブアプリからの利用が想定されていますが、上図中ではまとめて記載しています。

Webアプリ(ブラウザ)とネイティブアプリでは、細かいフローは異なりますが、本記事では簡略化しています。

以下の流れで説明します。
1. ユーザー登録
2. メールアドレス検証
3. ログイン
4. セッション取得
5. 認証が必要なAPIへのリクエスト

### ユーザー登録
ユーザー自身でのユーザー登録機能を備えています。

登録するIdentifierに相当する項目は、kratosに設定する[Identity schema](https://www.ory.sh/docs/kratos/manage-identities/customize-identity-schema)によって定義されており、上図のように、emailをIdentifierとして使用し、パスワードによる認証を行う場合は以下のような記述になります。

([Json schema](https://json-schema.org/)で記述されています。)

```json
{
  "email": {
    "type": "string",
    "format": "email",
    "title": "E-Mail",
    "ory.sh/kratos": {
      "credentials": {
        "password": {
          "identifier": true
        }
      },
      "verification": {
        "via": "email"
      },
      "recovery": {
        "via": "email"
      }
    }
  },
  "nickname": {
    "type": "string",
    "title": "nickname"
  },
  "birthdate": {
    "type": "string",
    "title": "birthdate"
  }
}
```

フォームからemail項目を入力し、kratosのRegistration APIを呼び出すと、上記のschemaに従ったidentityがkratosのDBに登録されます。

登録されたidentityに対して、ログイン試行することになりますが、EmailをIdentifierとして使用する場合は、メールアドレス検証が必要になる場合があります。

### traitsとその更新
上述のshchema、traitsと呼ばれ、ユーザーが自分自身で登録や更新が可能な項目です。

identifierであるemail以外にも、nicknameとbirthdateの項目があります。

これらは、認証のidentifierとして使用されるわけではないのですが、ユーザーに固有のプロファイル情報として、traitsに保管されます。

これらの項目は、ユーザー登録時に一緒に登録もできますし、traitsを更新可能な機能(sefl-service settings flow)も用意されていますので、任意のタイミングでユーザー自身で更新可能です。

(図中ではSelfService Settingsは割愛しています。)

### メールアドレス検証
EmailをIdentifierとして使用する場合、ユーザー登録後にメールアドレス検証プロセスが開始されます。

kratosの設定によって、メールアドレスが検証完了している場合でなければ、ログインさせないような設定も可能です。

基本的に、メールアドレスの検証が必須と考えた方が良いかと思います。

kartosはメール送信機能も内包されており、送信メッセージはkratos DBのcourier_messagesテーブルに格納されます。

メール内容はtemplateでカスタムすることが可能です。

templateはgo-templateが使用されており、テンプレートの中で概要ユーザーのIdentity情報を参照することができます。

登録したユーザーのメールアドレス宛に、6桁の検証コードが記載されたメールが届きます。

この検証コードをverification formへ入力し、kartosのVerification APIを呼び出します。

検証コードが正しく、また有効期限内であれば、メールアドレスの検証が完了します。

### ログイン
メールアドレス検証完了後に登録したユーザーのEmailとパスワードを入力し、kratosのLogin APIを呼び出します。

ログイン成功後、kratos DBのsessionsテーブルにセッションが格納されます。

### セッション取得
ログイン後に、whoamiエンドポイントからログインセッションを取得できます。

セッション管理方式は、Webアプリ(ブラウザ)とネイティブアプリで異なります。
* Webアプリ(ブラウザ)はCookieベースのセッション管理
* ネイティブアプリはトークンベースのセッション管理

ブラウザの場合は、セッションクッキーが付与されます。

Cookieであるため、特に意識ぜずとも以降のリクエストにセッションクッキーが付与されます。

ネイティブアプリの場合は、セッショントークンが返却されます。

Cookieと異なり、以降のAPIアクセス時のリクエストヘッダに、意識してセッショントークンを付与して呼び出す必要があります。

### 認証が必要なAPIへのリクエスト
上述の通り、ブラウザの場合ではセッションクッキー、ネイティブアプリの場合はヘッダにセッショントークンを付与して、APIを呼び出します。

API側は、ミドルウェア等でセッションが有効であるかどうかを確認し、有効であればAPIを実行し、そうでなければ401エラーを返却します。

APIからのセッションの有効性確認には、kratosのwhoamiエンドポイントを使用します。

(UIから実施する場合と同様のエンドポイントです。)

セッションが有効であれば、Identity情報も含んだSession情報が返却されます。

Session情報の例を以下に示します。

traitsやmetadataの

```json
{
  "session": {
    "id": "d56e59d8-742c-4ce1-a230-8cbf8cc00383",
    "active": true,
    "expires_at": "2024-01-30T02:33:34.377841929Z",
    "authenticated_at": "2024-01-29T02:33:34.377841929Z",
    "authenticator_assurance_level": "aal1",
    "authentication_methods": [
      {
        "method": "password",
        "aal": "aal1",
        "completed_at": "2024-01-29T02:33:34.377833887Z"
      }
    ],
    "issued_at": "2024-01-29T02:33:34.377841929Z",
    "identity": {
      "id": "20baf766-427e-4241-b6be-1cb95dfd2430",
      "schema_id": "user_v1",
      "schema_url": "http://localhost:4533/schemas/dXNlcl92MQ",
      "state": "active",
      "state_changed_at": "2024-01-23T14:07:33.475358Z",
      "traits": {
        "email": "user1@ysexample.com"
      },
      "verifiable_addresses": [
        {
          "id": "b3bee943-b9bc-4c75-93ad-fc351a709c7a",
          "value": "user1@ysexample.com",
          "verified": true,
          "via": "email",
          "status": "completed",
          "verified_at": "2024-01-24T10:00:43.604377Z",
          "created_at": "2024-01-23T14:07:33.47736Z",
          "updated_at": "2024-01-23T14:07:33.47736Z"
        }
      ],
      "recovery_addresses": [
        {
          "id": "ea8070ca-3177-477e-b28c-cdbb48284720",
          "value": "user1@ysexample.com",
          "via": "email",
          "created_at": "2024-01-23T14:07:33.478646Z",
          "updated_at": "2024-01-23T14:07:33.478646Z"
        }
      ],
      "metadata_public": null,
      "created_at": "2024-01-23T14:07:33.476249Z",
      "updated_at": "2024-01-23T14:07:33.476249Z"
    },
    "devices": [
      {
        "id": "c8ee6e7c-d77c-471b-afa3-2fdd17333d83",
        "ip_address": "192.168.65.1:38512",
        "user_agent": "curl/7.87.0",
        "location": ""
      }
    ]
  }
}
```

## kratosと周辺アーキテクチャ設計のTips

### kratos IdentityとアプリケーションDBのデータとの紐付け
kratosとアプリケーションAPIではDBが分かれています。

このため、kratos DBのIdentityをアプリケーションのDB内のユーザー固有の情報を紐付ける必要があります。

アプリケーションDBにユーザーに紐づくデータのテーブルのスキーマにkratosのIdentity IDを持たせるのがシンプルな方法です。

kratosのIdentity IDはUUID形式です。

kratosのDBには、MySQLとPostgreSQLをサポートしていますが、PostgreSQLにはUUID型が存在し、UUIDを扱いやすいです。

(CockroachDBもサポートしてますが、これにはあまり詳しくありません。)

MySQLでも文字列型でUUIDを扱うことはできますが、パフォーマンス的な懸念があります。

これについての詳しいところは以下の記事を参考にさせていただきました。

[MySQLでプライマリキーをUUIDにする前に知っておいて欲しいこと](https://techblog.raccoon.ne.jp/archives/1627262796.html)

MySQL8では、UUID_TO_BIN、BIN_TO_UUID関数がサポートされたため、BINALY型に変換して持つことで、パフォーマンス的な懸念は解消されそうですが、代わりにDBを参集する際に、ちょっと面倒さが残ります。

（BIN_TO_UUIDをSELECTに含めればいいだけではあるのですが、個人的にはそのちょっとがわりと面倒に感じてしまいます。）

### traitsに格納するべき項目

karatosのSelfService Settings Flowによって安全に更新可能であるため、traitsにはシステム全体で使用されるような項目を持たせるのが良いとのことです。

traitsに格納するべきデータは、ベストプラクティスによると

> Do not add too many fields to your identities. Keep the number of fields well below 15.

IDにフィールドを増やしすぎないこと。フィールドの数は15以下にしましょう。

> Do not store business logic in your identities. Store this information in other systems. This includes credit card information, shipping addresses, shopping cart items, or user preferences.

ビジネス・ロジックをアイデンティティに保存しないでください。これらの情報は他のシステムに保存してください。これには、クレジットカード情報、配送先住所、ショッピングカート項目、またはユーザー設定が含まれます。

> Do store profile data that is used across your system. This includes the usernames, email addresses, phone numbers, first names, and last names.

システム全体で使用されるプロファイルデータを保存してください。これには、ユーザー名、メールアドレス、電話番号、姓、名が含まれます。

とのことです。

ビジネスロジックに関連しない、ユーザーのプロファイルに関する項目はtraitsに持たせ、それ以外はアプリケーションDB側に持たせるイメージですね。

### metadata
ちなみに、metadata_public, metadata_adminには、ユーザーが自身で更新不可能な項目を持たせることができます。

metadata_publicは、ログインセッションに含まれる情報で、ユーザーが参照可能です。

一方、metadta_adminは、ログインセッションに含まれておらず、ユーザーが参照が不可能な項目です。

例として、metadata_publicにroleという項目を持たせ、ロールによって認可制御を行うなどの実装が考えられます。

この場合、ユーザー自身ではロールは設定できない（させたくない）ため、アドミン権限を持つ管理者ユーザーがkratosのAdmin系APIを使用して、metadata_publicを更新することになります。

## 一旦まとめ
ID管理および認証機能導入の選択肢および、複数のAPIが構築される状況下で、ory kratos選択した背景、理由について説明しました。

kratosの一通りの機能やユースケース概要、設計Tipsについても紹介しました。

私自身、2つほどのプロジェクトでory kratosを導入していまして、最初の使い方のキャッチアップは少し大変でしたが、そこを乗り越えてしまえば便利に使用できる印象です。

NIST準拠のセキュリティ観点で実装されていたり、Golangベースかつ状態管理の依存先がDBのみとシンプルな作りでもあるため、スケーリングやパフォーマンス観点でも使いやすいです。

まだまだ、全ての機能を使い込んでいるわけではないのですが、今後もkratosを使い込み、使い方や実装サンプルについても紹介していきたいと思います。

## SelfService flowのサンプル
早速ですが、以下のリポジトリにkratosのself-service flowのごく簡単なサンプルを作成しました。

https://github.com/YoshinoriSatoh/kratos_selfservice_example

docker compose upで、kratos、DB, メールサーバのmailslurperが起動します。

```
docker compose up
```

kratosに対して、SelfService flowの各種操作を実行するbashスクリプトを用意しています。

詳細は省きますが、一部だけざっと紹介します。

### Registration flow (browser)

例えば、Browser flowのユーザー登録は以下のように実行します。

```
./scripts/registration_browser.sh
```

上記スクリプト中で、以下が実行されています。
1. Registration flowの初期化API
2. Registration flowの実行API
3. Verification code入力待ち
4. Verification flow(mothod: code)の実行API 


スクリプトを実行すると、1と2まで実行され、3で以下のようなプロンプトが表示されます。

```
please input code emailed to you:  
```

ここで、ブラウザで http://localhost:4436 にアクセスしてメールを確認します。

![](https://github.com/YoshinoriSatoh/zenn/blob/master/images/kartos_usecase_overview/mailslurper.png?raw=true)

メールアドレス検証のための検証コードが送信されています。

「Please verify your email address」というメールが届いています。

メール本文中に、6桁の検証コードが記載されています。

この検証コードをプロンプトに入力します。

入力すると、4のflowの実行APIが実行されます。

ステータスが200 OKが返却されればOKです。

### Login flow (browser)

次に、登録したユーザーでログインしてみます。

```
./scripts/login_browser.sh
```

上記スクリプト中で、以下が実行されています。
1. Login flowの初期化API
2. Login flowの実行API

実行後、2.のレスポンスが以下のように返却されます。

レスポンスボディにsessionが返却されると同時に、browser flowであるため、session cookieも付与されます。

bashスクリプトでは、cookieはファイルで保管しており、`.cookie_session`というファイル内に、`kratos_session`というキーのcookieが保管されます。

```json
{
  "session": {
    "id": "2eb50f70-3a23-4067-99f6-9fd645686880",
    "active": true,
    "expires_at": "2024-01-30T13:53:23.622162292Z",
    "authenticated_at": "2024-01-29T13:53:23.622162292Z",
    "authenticator_assurance_level": "aal1",
    "authentication_methods": [
      {
        "method": "password",
        "aal": "aal1",
        "completed_at": "2024-01-29T13:53:23.6221575Z"
      }
    ],
    "issued_at": "2024-01-29T13:53:23.622162292Z",
    "identity": {
      "id": "eedb3b8d-3ecf-4946-9f7b-440f65ea5fca",
      "schema_id": "user_v1",
      "schema_url": "http://localhost:4533/schemas/dXNlcl92MQ",
      "state": "active",
      "state_changed_at": "2024-01-29T12:37:10.960541Z",
      "traits": {
        "email": "1@local"
      },
      "verifiable_addresses": [
        {
          "id": "5993eb3c-34e9-444b-930c-1234df761b59",
          "value": "1@local",
          "verified": true,
          "via": "email",
          "status": "completed",
          "verified_at": "2024-01-29T13:36:18.727252Z",
          "created_at": "2024-01-29T12:37:10.964533Z",
          "updated_at": "2024-01-29T12:37:10.964533Z"
        }
      ],
      "recovery_addresses": [
        {
          "id": "61e32d2c-ff31-46d9-9c26-70da154ebdc4",
          "value": "1@local",
          "via": "email",
          "created_at": "2024-01-29T12:37:10.965891Z",
          "updated_at": "2024-01-29T12:37:10.965891Z"
        }
      ],
      "metadata_public": {
        "roles": [
          "admin"
        ]
      },
      "created_at": "2024-01-29T12:37:10.962889Z",
      "updated_at": "2024-01-29T12:37:10.962889Z"
    },
    "devices": [
      {
        "id": "83ec820f-20b7-4f52-95b2-ebf184da3ecb",
        "ip_address": "192.168.65.1:55593",
        "user_agent": "curl/7.87.0",
        "location": ""
      }
    ]
  }
}
```

cookieファイルの中身は以下のようになっています。
```
# Netscape HTTP Cookie File
# https://curl.se/docs/http-cookies.html
# This file was generated by libcurl! Edit at your own risk.

#HttpOnly_.localhost	TRUE	/	FALSE	1706622802	kratos_session	MTcwNjUzNjQwM3xRZWpxSXExVVl1ZlpncGc3VEZoQTRwcklrLWJpTk9tR2xUTE1idXpRQ2xEbWpIX1VITjA1ZkhHdmttZDVEWURfYUE3SXU5dkFRNmQ3dmJwQ2pTbEhHbTJla1MzRWk0TV9UY2xfS095Mm5aVFNIUm1famVESExsd01nbHlGV2lQT3pBSzB3eUlRd3Vsa3dkaFE0VUdXa3lKRFItQVU5dkM1bXIxWDl2dkZET0dZR2JEWkI3ckdIOEk0ZUxVdy14ZEg2WlNiaXNWZVdUR0puSXJGanIyOHZqZS1jYXY2RFNHZHZCN2FZX2ZKWnZyTXpqWGVuZ0JjTUE3TEFaQjRlSHhVVkFMbFV6VFJ2Y2UwR0wySlRybVRwdz09fMUIK6aQEukbx3UmWsmj7Y9Vhs4QYO2vG5aZCmMSMGbF
#HttpOnly_.localhost	TRUE	/	FALSE	1738072403	csrf_token_0c666b0902e91df80a1a31234b66208e51e130bbc39abed17c5efa1830e47544	3myyvq+HAIPBmCi78b8BqcwgtOT50lkquTg6Nk+Bwu0=

```

### whoami (browser)

上記で取得したkratos_sessionから、セッション情報を取得するには、/sessions/whoamiエンドポイントを呼び出します。

/sessions/whoamiエンドポイントは、SPAにおけるUI側からも呼び出せますし、API側でcookieが付与されたリクエストのセッションが有効なものであるかを確認するためにも使用します。

```
./scripts/whoami_browser.sh
```

レスポンスには、Session情報が返却されます。(ログインのレスポンスと殆ど同様)

```json
{
  "id": "2eb50f70-3a23-4067-99f6-9fd645686880",
  "active": true,
  "expires_at": "2024-01-30T13:53:23.622162Z",
  "authenticated_at": "2024-01-29T13:53:23.622162Z",
  "authenticator_assurance_level": "aal1",
  "authentication_methods": [
    {
      "method": "password",
      "aal": "aal1",
      "completed_at": "2024-01-29T13:53:23.6221575Z"
    }
  ],
  "issued_at": "2024-01-29T13:53:23.622162Z",
  "identity": {
    "id": "eedb3b8d-3ecf-4946-9f7b-440f65ea5fca",
    "schema_id": "user_v1",
    "schema_url": "http://localhost:4533/schemas/dXNlcl92MQ",
    "state": "active",
    "state_changed_at": "2024-01-29T12:37:10.960541Z",
    "traits": {
      "email": "1@local"
    },
    "verifiable_addresses": [
      {
        "id": "5993eb3c-34e9-444b-930c-1234df761b59",
        "value": "1@local",
        "verified": true,
        "via": "email",
        "status": "completed",
        "verified_at": "2024-01-29T13:36:18.727252Z",
        "created_at": "2024-01-29T12:37:10.964533Z",
        "updated_at": "2024-01-29T12:37:10.964533Z"
      }
    ],
    "recovery_addresses": [
      {
        "id": "61e32d2c-ff31-46d9-9c26-70da154ebdc4",
        "value": "1@local",
        "via": "email",
        "created_at": "2024-01-29T12:37:10.965891Z",
        "updated_at": "2024-01-29T12:37:10.965891Z"
      }
    ],
    "metadata_public": {
      "roles": [
        "admin"
      ]
    },
    "created_at": "2024-01-29T12:37:10.962889Z",
    "updated_at": "2024-01-29T12:37:10.962889Z"
  },
  "devices": [
    {
      "id": "83ec820f-20b7-4f52-95b2-ebf184da3ecb",
      "ip_address": "192.168.65.1:55593",
      "user_agent": "curl/7.87.0",
      "location": ""
    }
  ]
}
```

### おわりに
以上、ory kratosとその実装サンプルの紹介でした。

今後もkratosの機能やその使い方について、発信していきたいと思います。

(Passkey実装されてほしいなぁ)
