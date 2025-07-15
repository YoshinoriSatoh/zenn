---
title: HTMX を使用した kratos browser flow の実装サンプル
emoji: ":lock:" 
type: "tech" 
topics: ["ory", "kratos", "authentication", "htmx", "golang"] 
published: false
---  

## 本記事の概要
以下の記事で、[ory kratos](https://www.ory.sh/docs/welcome)のユースケース概要と、shell scriptを使用した簡単な実行サンプルを紹介しました。

@[card](https://zenn.dev/yoshinori_satoh/articles/kartos_usecase_overview)

本記事では、ブラウザからkratosの各種self-service flowを実行する、golang と[HTMX](https://htmx.org/)を使用したサンプルコードの紹介と一部抜粋して解説します。

## サンプルコード

サンプルコードは以下のリポジトリです。

@[card](https://github.com/YoshinoriSatoh/kratos_example/tree/kratos_v1_1_0_selfservice)

解説は後述します。

## self-service flow
改めてself-service flowについて簡単に説明します。

kratosには、ユーザー自身によるユーザー登録やログイン、アカウント復旧といった、[SelfService flow](https://www.ory.sh/docs/kratos/self-service)と呼ばれる実装がなされており、この仕様はNISTやIFTF、Microsoft Research、Google Research、Trou Huntによって確立されたベストプラクティスに基づいているとのことです。

self-ervice flowに従うことで、各種攻撃やCSRFに対するセキュリティが確保されます。

self-service flowの流れは、以下のドキュメントに記載されています。

https://www.ory.sh/docs/kratos/self-service#browser-flows-for-server-side-apps-nodejs-php-java-

self-service flowの実装方式には、以下3つの方法があります。
* Browser-based flows
  * Browser-based flows for server-side apps
  * Browser-based flows also support client-side applications
* API flows

Webアプリであれば、`Browser-based flows`、ネイティブアプリであれば`API flows`を使用します。

最も大きな違いはセッションの管理方法で、`Browser-based flows`はCookieを使用しますが、`API flows`はTokenを使用します。

また、Webアプリの場合で、ReactやVueのようなSPAの場合は`Browser-based flows also support client-side applications`を使用します。

この場合、kratosへのAPIリクエスト時に、`Accept: application/json` ヘッダを付与し、結果をJSONで受け取り、ダイレクトは発生しません。

SPAを使用せず、PHPやJava等のMVCフレームワーク等を使用してサーバサイドでHTMLをレンダリングする場合は`Browser-based flows for server-side apps`を使用します。

この場合は、kratosへのAPIリクエスト時に、`Accept: text/html` ヘッダを付与し、結果をHTMLで受け取り、またリダイレクトが返却されることもあります。

但し、本記事のサンプルでは、サーバサイドでHTMLをレンダリングしているものの、HTMXを使用した[HDA(Hypermedia-Driven Applications)](https://htmx.org/essays/hypermedia-driven-applications/)であるため、上記のユースケースに該当しません。

そこで、ブラウザからkratosへ直接アクセスするのではなく、APIを経由してアクセスし、kratosから返却された情報をHTMXを使用してレンダリングする構成としています。

APIからkaratosへアクセスする際には、JSONでやりとりすることになるため、`Browser-based flows also support client-side applications`を使用しています。

APIからkratosを使用しているため、一見、`API flows`を使用するべきようにも見えますが、`API flows`はあくまでも、Cookieを利用不可能なネイティブアプリに使用することを想定したものです。

構成の詳細は後述します。

### flowの種類と実行の流れ

![](https://github.com/YoshinoriSatoh/zenn/blob/master/images/kratos_browser_flow_example/kratos_flow_overview.png?raw=true)

flowには以下の種類があります。

* Registration flow (ユーザーの登録)
* Verification flow (メールアドレス検証)
* Login flow (ログイン)
* Recovery flow (パスワードリセット)
* Settings flow (プロフィール更新/パスワード変更)

基本的には、最初にflowを作成し、発行されたflowに所定のパラメータを指定して、flowを更新します。

kratosには専用のDBが存在し、flow情報はDBに保管されます。

`ui`カラムには、レンダリングに必要な情報が格納され、create時とupdate時の結果によって内容が更新されます。

idを指定してflowを取得するAPIも用意されており、画面遷移等の都合でflow情報を取得し、`csrf_token`等の必要な情報を取得することがあります。

browser flwoの場合にのみ使用する、`csrf_token`も格納されています。

flowには有効期限`expires_at`があります。

flowによっては、状態を持ち、更新ステップが2回必要なものもあります。

flowの情報は、flowの種類ごとにkratosのDBの`selfservice_xxxxxx_flows`テーブルに保管されており、flowの状態は`state`カラムに保管されています。

### flow間の遷移

Registration flow と Recovery flowについては、flowの完了時点で別のflowが作成されます。

#### Registration flow から Verification flow への遷移

ユーザー登録時に、Registration flowが使用されます。

Identity Schemaの IdentifierにEmailが存在し、なおかつEmailを使用してVeificationを実行するように指定している場合は、Registration flow完了後に、Emailを検証するためのVerification flowが作成され、検証メールの送信までが実施されます。

```json:kratos/general/identity.schema.user_v1.json
 "properties": {
    "traits": {
      "type": "object",
      "properties": {
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
            ...
          }
        },
        ...
```

![](https://github.com/YoshinoriSatoh/zenn/blob/master/images/kratos_browser_flow_example/registration_flow_move.png?raw=true)

通常、Verification flowの作成をkratos APIを通じて行った場合、Verification flow のstateは`choose_method`になり、検証対象のEMailによる更新を待つ状態となりますが、Registration flow完了後はこのステップまでが実施された状態となり、Verification flowのstateも`sent_email`に更新されます。

この後は、検証メールに記載された検証コードを使用して、Verification flowを更新すると、stateが`passed_challenge`となり、Verification flowが完了します。

#### Recovery flow から Settings flow への遷移

![](https://github.com/YoshinoriSatoh/zenn/blob/master/images/kratos_browser_flow_example/recovery_flow_move.png?raw=true)

パスワードリセット時に、Recovery flowが使用されます。

メールアドレスを入力してRecovery flowを作成し、送信された復旧コードを使用して、Recovery flowが完了すると、パスワードを設定するためのSettings flowが作成されます。

Recovery flow完了時にセッションも発行されるため、作成されたSettings flowを使用して、パスワードを設定することができます。

## サンプルの構成

改めて、サンプルコードのリポジトリは以下です。

@[card](https://github.com/YoshinoriSatoh/kratos_example/tree/kratos_v1_1_0_selfservice)

![](https://github.com/YoshinoriSatoh/zenn/blob/master/images/kratos_browser_flow_example/browser_flow_impl_structure.png?raw=true)

Golang実装のAPIサーバーと、kratos、およびkratosのDBが起動します。

kratosからメール送信を行うため、ローカル確認用のメールサーバであるMailSlurperも起動しています。

### 起動コマンド
```zsh
docker compose up
```

### 起動するコンテナ
docker compose で以下のコンテナが起動します。

| コンテナ | 外部公開エンドポイント | 概要 |
| ---- | ---- | ---- |
| kratos | - | kratosサーバー |
| kratos-migrate | - | kratosのDBマイグレーション(実行後終了) |
| db-kratos | postgres://kratos:secret@db-kratos:5432/kratos | kratosで使用する Postgres |
| mailslurper | http://localhost:4436 | ローカル用メールサーバ |
| app-sample | http://localhost:3000 | kratosを使用したサンプルアプリケーション |


### ディレクトリ構成
```
.
├── app // サンプルアプリケーションのAPI(Golang)
│   └── sample
│       ├── Dockerfile
│       └── cmd
│         └── server // サンプルアプリケーションのエントリーポイント
│       ├── go.mod
│       ├── go.sum
│       ├── handler // HTTP Request/Responseのハンドリング
│       ├── kratos  // kratos API呼び出しをラッピング
│       ├── static // 静的ファイル
│       └── templates // HTMLテンプレート
└── kratos 
    ├── config.yml // kratos設定ファイル
    ├── identity.schema.user_v1.json // Identity Schema
    └── templates // メールテンプレート
```

### HTMXによるHDAの採用
本記事のサンプルでは、golangでAPIサーバを実装し、APIサーバを経由してkratosへアクセスする構成としています。

![](https://github.com/YoshinoriSatoh/zenn/blob/master/images/kratos_browser_flow_example/browser_flow_impl_overview.png?raw=true)

ReactやVueのようなSPAを使用しないサーバサイドレンダリングですが、HTMXによる[HDA(Hypermedia-Driven Applications)](https://htmx.org/essays/hypermedia-driven-applications/)を採用しています。

HDAをざっくり説明すると、HTMLの一部分(フラグメント)をレンダリングして返却するが、ページ全体を読み込み直すことなく部分的に描画可能な方法です。

最初のURLアクセスでは、ページ全体のHTMLが返却され（通常のサーバサイドレンダリング）、読み込んだページの中で非同期でAPIを呼び出すと(AJAX)、HTMLのフラグメントが返却されます。

このように、サーバサイドレンダリングとクライアントサイドレンダリングが合成されたような仕組みがHDAです。

今後の開発で効率が良くなりそうと考え、実験的にHDA(HTMX)を採用しています。

### HDAを前提としたBrowser-based flowsの実装方針

HDAの仕組み上、ブラウザからkratosへ直接アクセスする方式には少々不都合があります。

まず、`Browser-based flows also support client-side applications`では、返却値がJSONであるため、そのまま直接は使用できません。

また、`Browser-based flows for server-side apps`では、エラー時等の画面遷移の際に、kratosからリダイレクトが返却されます。

リダイレクト先の処理で、改めてflowやエラー情報を取得し、レンダリングしたHTMLを返却する実装も可能ですが、HDA的にあまり直感的でないように思いました。

非同期でAPIを呼び出してもしエラーが発生した場合、エラー情報も含めてレンダリングされたHTMLが返却されて欲しいところです。

HDAでもリダイレクトによる画面遷移を使用することはありますが、処理の内容によって、リダイレクトするか、HTMLフラグメントを返却するかは選択できる方が直感的な制御ができると思います。

上記を考慮すると、kratosへのアクセスはAPIサーバを経由し、APIサーバからレンダリングされたHTMLを返却する構成が最適と考えました。

APIサーバーからkratosへアクセスする際には、JSONでやりとりしたいため`Browser-based flows also support client-side applications`を使用しています。

APIサーバー内で、kratos SDKを使用して実装しており、SDK使用時には`Accept: application/json`が付与されており、自ずと`Browser-based flows also support client-side applications`が使用されます。

## ユーザー登録
ユーザー登録から、メールアドレスの検証までの流れを解説します。

Registration flow -> Verification flowの流れで実行します。

### Registration flowの作成と更新

ユーザー登録は、最初にRegistration flowを作成します。

```go:app/auth-general/handler/handler_auth.go
type handleGetAuthRegistrationdRequestParams struct {
	cookie string
	flowID string
}

func (p *Provider) handleGetAuthRegistration(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	session := getSession(ctx)

	reqParams := handleGetAuthRegistrationdRequestParams{
		cookie: r.Header.Get("Cookie"),
		flowID: r.URL.Query().Get("flow"),
	}

	// Registration Flow の作成 or 取得
	// Registration flowを新規作成した場合は、FlowIDを含めてリダイレクト
	output, err := p.d.Kratos.CreateOrGetRegistrationFlow(kratos.CreateOrGetRegistrationFlowInput{
		Cookie: reqParams.cookie,
		FlowID: reqParams.flowID,
	})
	if err != nil {
		w.WriteHeader(http.StatusOK)
		tmpl.ExecuteTemplate(w, templatePaths.AuthRegistrationIndex, viewParameters(session, r, map[string]any{
			"ErrorMessages": output.ErrorMessages,
		}))
		return
	}
  ...続く
```

kratosへのアクセスは`kratos`パッケージを使用してラッピングしています。

```go:app/auth-general/kratos/selfservice.go 
type CreateOrGetRegistrationFlowInput struct {
	Cookie string
	FlowID string
}

type CreateOrGetRegistrationFlowOutput struct {
	Cookies       []string
	FlowID        string
	IsNewFlow     bool
	CsrfToken     string
	ErrorMessages []string
}

// Registration Flow がなければ新規作成、あれば取得
// csrfTokenは、本来は *kratosclientgo.RegistrationFlow から取得できるはずだが、
// kratos-client-go:v1.0.0 に不具合があるため、http.Response から取得し返却している
func (p *Provider) CreateOrGetRegistrationFlow(i CreateOrGetRegistrationFlowInput) (CreateOrGetRegistrationFlowOutput, error) {
	var (
		err              error
		response         *http.Response
		registrationFlow *kratosclientgo.RegistrationFlow
		output           CreateOrGetRegistrationFlowOutput
	)

	// flowID がない場合は新規にRegistration Flow を作成
	// flowID がある場合はRegistration Flow を取得
	if i.FlowID == "" {
		registrationFlow, response, err = p.kratosPublicClient.FrontendApi.
			CreateBrowserRegistrationFlow(context.Background()).
			Execute()
		if err != nil {
			slog.Error("CreateRegistrationFlow Error", "RegistrationFlow", registrationFlow, "Response", response, "Error", err)
			output.ErrorMessages = getErrorMessages(err)
			return output, err
		}
		slog.Info("CreateRegistrationFlow Succeed", "RegistrationFlow", registrationFlow, "Response", response)

		output.IsNewFlow = true

	} else {
		registrationFlow, response, err = p.kratosPublicClient.FrontendApi.
			GetRegistrationFlow(context.Background()).
			Id(i.FlowID).
			Cookie(i.Cookie).
			Execute()
		if err != nil {
			slog.Error("GetRegistrationFlow Error", "RegistrationFlow", registrationFlow, "Response", response, "Error", err)
			output.ErrorMessages = getErrorMessages(err)
			return output, err
		}
		slog.Info("GetRegisrationFlow Succeed", "RegistrationFlow", registrationFlow, "Response", response)
	}

	output.FlowID = registrationFlow.Id

	// SDKを使用しているので、本来は上記レスポンスの第一引数である
	// *kratosclientgo.RegistrationFlow から csrf_token その他を取得するところだが、
	// goのv1.0.0のSDKには不具合があるらしく、仕方ないのでhttp.Responseのbodyから取得している
	// https://github.com/ory/sdk/issues/292
	b, err := readHttpResponseBody(response)
	if err != nil {
		slog.Error(err.Error())
		return output, err
	}
	output.CsrfToken = getCsrfTokenFromResponseBody(b)

	// browser flowでは、kartosから受け取ったcookieをそのままブラウザへ返却する
	output.Cookies = response.Header["Set-Cookie"]

	return output, nil
}
```

FlowIDの指定がなければ新規にRegistration Flowを作成し、あれば作成済みのRegistration Flowを取得します。

Registration Flowには`CSRF Token`が含まれており、`Browser-based flows`では、flowの更新時にCSRF Tokenが必要となるため、取得しています。

kratosへのアクセスには[kratos-client-go](https://github.com/ory/kratos-client-go)のSDKに[不具合](https://github.com/ory/sdk/issues/292)があるようなので、http.Responseから取得しています。

また、`Browser-based flows`では、kratosからCookieも返却され、flowの作成時に、CSRF用のCookieも作成されています。

`Browser-based flows`でのflow更新時には、CSRF TokenとCSRF Cookieの両方が必要です。

kratosへ直接アクセスする場合は、kratosから返却されたCookieがそのままブラウザで読み込まれるため([withCredentials有効時](https://developer.mozilla.org/ja/docs/Web/API/XMLHttpRequest/withCredentials))、意識する必要はありませんが、本サンプルのようにAPIサーバーを経由する場合は、kratosから返却されたCookieを、明示的にAPIサーバーからブラウザへ返却する必要があります。

作成したRegistration Flowの`FlowID`と`CsrfToken`を埋め込み、HTMLをレンダリングします。

```go:app/auth-general/handler/handler_auth.go
  ...続き
	// kratosのcookieをそのままブラウザへ受け渡す
	setCookieToResponseHeader(w, output.Cookies)

	if output.IsNewFlow {
		redirect(w, r, fmt.Sprintf("%s?flow=%s", routePaths.AuthRegistration, output.FlowID))
		return
	}

	// flowの情報に従ってレンダリング
	w.WriteHeader(http.StatusOK)
	tmpl.ExecuteTemplate(w, templatePaths.AuthRegistrationIndex, viewParameters(session, r, map[string]any{
		"RegistrationFlowID": output.FlowID,
		"CsrfToken":          output.CsrfToken,
	}))
}
```

```html:app/sample/templates/auth/registration/_form.html
<form 
  id="registration-form"
  hx-post="/auth/registration?flow={{.RegistrationFlowID}}" 
  hx-swap="outerHTML" 
  hx-target="this"
>
  <input
    name="csrf_token"
    type="hidden"
    value="{{.CsrfToken}}"
  />

  <div class="mt-2 mb-4">
    <label class="form-control">
      <div class="label">
        <span class="label-text font-semibold">メールアドレス</span>
      </div>
      <input 
        id="email"
        name="email" 
        value="yoshinori.satoh.tokyo@gmail.com"
        placeholder="例) niko-chan@kratos-example.com"
        {{if .ValidationFieldError.Email}}
        class="input input-bordered input-error"
        {{else}}
        class="input input-bordered"
        {{end}}
      >
      {{if .ValidationFieldError.Email}}
      <div class="text-sm text-red-700 my-2">{{.ValidationFieldError.Email}}</div>
      {{end}}
    </label>

    <label class="form-control">
      <div class="label">
        <span class="label-text">パスワード</span>
      </div>
      <input 
        id="password"
        type="password" 
        name="password" 
        value="Overwatch2024!@"
        {{if .ValidationFieldError.Password}}
        class="input input-bordered input-error"
        {{else}}
        class="input input-bordered"
        {{end}}
      >
      {{if .ValidationFieldError.Password}}
      <div class="text-sm text-red-700 my-2">{{.ValidationFieldError.Password}}</div>
      {{end}}
    </label>
  </div>
	...
	<div class="mx-auto text-center">
    <button class="btn btn-primary btn-wide">次へ</button>
  </div>
```

HTMXでは、`hx-post`のような記述で、formの場合はsubmitをトリガーにAJAX通信が行われます。

クエリパラメータの`flow`と、hiddenの`csrf_token`、および各種input要素の`email`と`password`がbodyに含まれて送信されます。

上記の場合`POST /auth/registration`へAJAXでアクセスし、正常にflowを更新できればVerification flowのコード入力画面へ遷移し、もしエラーがあればレスポンスとしてHTMLフラグメントが返却されます。

### Registration flowの更新後のVerfiication flowへの遷移

上記のPOSTによって、Registration flowの更新が完了すると、検証メールが送信され、Verification flow が state=`sent_email`で作成され([上図参照](https://zenn.dev/yoshinori_satoh/articles/kratos_browser_flow_example#registration-flow-%E3%81%8B%E3%82%89-verification-flow-%E3%81%B8%E3%81%AE%E9%81%B7%E7%A7%BB))、またVerification flow IDが返却されます。

```go:app/auth-general/kratos/selfservice.go 
func (p *Provider) UpdateRegistrationFlow(i UpdateRegistrationFlowInput) (UpdateRegistrationFlowOutput, error) {
	var output UpdateRegistrationFlowOutput

	// Registration Flow の送信(完了)
	updateRegistrationFlowBody := kratosclientgo.UpdateRegistrationFlowBody{
		UpdateRegistrationFlowWithPasswordMethod: &kratosclientgo.UpdateRegistrationFlowWithPasswordMethod{
			Method:   "password",
			Password: i.Password,
			Traits: map[string]interface{}{
				"email": i.Email,
			},
			CsrfToken: &i.CsrfToken,
		},
	}
	successfulRegistration, response, err := p.kratosPublicClient.FrontendApi.
		UpdateRegistrationFlow(context.Background()).
		Flow(i.FlowID).
		Cookie(i.Cookie).
		UpdateRegistrationFlowBody(updateRegistrationFlowBody).
		Execute()
	if err != nil {
		slog.Error("UpdateError", "Response", response, "Error", err)
		output.ErrorMessages = getErrorMessages(err)
		return output, err
	}
	slog.Info("UpdateRegisrationFlow Succeed", "SuccessfulRegistration", successfulRegistration, "Response", response)

	// SDKを使用しているので、本来は上記レスポンスの第一引数である
	// *kratosclientgo.SuccessfulNativeRegistration から必要な値を取得するところだが、
	// goのv1.0.0のSDKには不具合があるらしく、仕方ないのでhttp.Responseから取得している
	// https://github.com/ory/sdk/issues/292
	b, err := readHttpResponseBody(response)
	if err != nil {
		slog.Error(err.Error())
		return output, err
	}
	output.VerificationFlowID = getContinueWithVerificationFlowId(b)

	// browser flowでは、kartosから受け取ったcookieをそのままブラウザへ返却する
	output.Cookies = response.Header["Set-Cookie"]

	return output, nil
}
```
Verification flow ID取得の際もCsrf Tokenの場合と同様に、kratos-client-goのSDKに不具合があるようなので、http.Responseから取得しています。

ここで、少しおさらいで、Registration flowからVerification flowへの遷移の流れ図を再掲します。

![](https://github.com/YoshinoriSatoh/zenn/blob/master/images/kratos_browser_flow_example/registration_flow_move.png?raw=true)

Verification flowには以下のステップがあり、stateが更新されます。

1. Verification flowの作成 (state -> choose_method) 
2. 検証対象のEmailを指定してflowを更新 (state -> sent_email)
3. 2.で指定したEmailへ送信された検証コードを指定してflowを更新 (state -> passed_challenge)

Registration flow完了時点では、kratos側で上記の手順2.までが実行されています。

ここで検証対象のEmailとは、ユーザー登録しようとしているEmailです。

残る手順は3.のみであり、Emailへ送信された検証コードと、Registration flow完了時に返却されたVerification flow IDを使用して、Verification flowを更新します。

### Verification flowの更新 (state -> passed_challenge)

Registration flow完了時に返却されたVerification flow IDを使用して、検証コード入力フォームをレンダリングします。

![](https://github.com/YoshinoriSatoh/zenn/blob/master/images/kratos_browser_flow_example/verification_code_input.png?raw=true)

HTMLテンプレートの一部を抜粋します。

```html:app/auth-general/templates/verification/_code_form.html
<form 
  id="verification-form" 
  hx-post="/auth/verification/code?flow={{.VerificationFlowID}}"
  hx-swap="outerHTML" 
  hx-target="this"
  > 
  <input
    name="csrf_token"
    type="hidden"
    value="{{.CsrfToken}}"
  />

  <div class="mt-2 mb-4">
    <label class="form-control">
      <div class="label">
        <span class="label-text">検証コード</span>
      </div>
      <input 
        name="code" 
        min="6" 
        max="6" 
        {{if .ValidationFieldError.Code}}
        class="input input-bordered input-error"
        {{else}}
        class="input input-bordered"
        {{end}}
      />
      {{if .ValidationFieldError.Code}}
      <div class="text-sm text-red-700 my-2">{{.ValidationFieldError.Code}}</div>
      {{end}}
    </label>
  </div>

  <div class="mx-auto text-center">
    <button class="btn btn-primary btn-wide">送信</button>
  </div>

  {{ template "_alert.html" }}
</form>
```

Registration flowと同様、上記HTMLには、`VerificationFlowID`と`CsrfToken`が埋め込まれています。

formがsubmitされると、`POST /auth/verification/code`がAJAXでアクセスされ、正常にflowを更新できればVerification flowを完了してログイン画面へ、もしエラーがあればレスポンスとしてHTMLフラグメントが返却されます。


```go:app/auth-general/kratos/selfservice.go UpdateVerificationFlow
func (p *Provider) UpdateVerificationFlow(i UpdateVerificationFlowInput) (UpdateVerificationFlowOutput, error) {
	var (
		output     UpdateVerificationFlowOutput
		updateBody kratosclientgo.UpdateVerificationFlowWithCodeMethod
	)

	// email設定時は、Verification Flowを更新して、アカウント検証メールを送信
	// code設定時は、Verification Flowを完了
	if i.Email != "" && i.Code == "" {
		updateBody = kratosclientgo.UpdateVerificationFlowWithCodeMethod{
			Method:    "code",
			Email:     &i.Email,
			CsrfToken: &i.CsrfToken,
		}
	} else if i.Email == "" && i.Code != "" {
		updateBody = kratosclientgo.UpdateVerificationFlowWithCodeMethod{
			Method:    "code",
			Code:      &i.Code,
			CsrfToken: &i.CsrfToken,
		}
	} else {
		err := fmt.Errorf("parameter convination error. email: %s, code: %s", i.Email, i.Code)
		slog.Error("Parameter convination error.", "email", i.Email, "code", i.Code)
		return output, err
	}

	// Verification Flow の送信(完了)
	updateVerificationFlowBody := kratosclientgo.UpdateVerificationFlowBody{
		UpdateVerificationFlowWithCodeMethod: &updateBody,
	}
	successfulVerification, response, err := p.kratosPublicClient.FrontendApi.
		UpdateVerificationFlow(context.Background()).
		Flow(i.FlowID).
		Cookie(i.Cookie).
		UpdateVerificationFlowBody(updateVerificationFlowBody).
		Execute()
	if err != nil {
		slog.Error("UpdateVerificationFlow Error", "Response", response, "Error", err)
		output.ErrorMessages = getErrorMessages(err)
		return output, nil
	}
	slog.Info("UpdateVerification Succeed", "SuccessfulVerification", successfulVerification, "Response", response)

	// browser flowでは、kartosから受け取ったcookieをそのままブラウザへ返却する
	output.Cookies = response.Header["Set-Cookie"]

	return output, nil
}
```

flow ID, CSRF Tokenおよび検証コードを受け取り、Verification flowを更新しています。

更新に成功すると、stateは`passed_challenge`に更新されます。

Registration flow から Verification flow へ遷移する場合だけでなく、Verification flowを最初から実行するケースもあります。

(登録後の画面から検証コード入力し損ねて、別画面へ遷移してしまった場合など)

このため、Emailが指定された場合は、

`2. 検証したいEmailを使用して、flowを更新 (state -> sent_email)`を、

Codeが指定された場合は

`3. 2.で指定したEmailへ送信された、検証コードを使用して、flowを更新 (state -> passed_challenge)`

を実行します。

## アカウント復旧(パスワードリセット) 
パスワードリセット時の流れを解説します。

Recovery flow -> Settings flow (password)の流れで実行します。

### Recovery flowの作成と更新

Recovery flowの更新は、以下の2ステップ必要です。
ここでも、Recovery flowからSettings flowへの遷移の流れ図を再掲します。

![](https://github.com/YoshinoriSatoh/zenn/blob/master/images/kratos_browser_flow_example/recovery_flow_move.png?raw=true)

Verification flowには以下のステップがあり、stateが更新されます。

1. Recovery flowの作成 (state -> choose_method)
2. アカウント復旧したいメールアドレスを指定して更新 (state -> sent_email)
3. メールアドレスに送信されたアカウント復旧コードを指定して更新 (state -> passed_challenge)

Recovery flowの作成は、[Registration flowの作成と同様です。](https://github.com/YoshinoriSatoh/kratos_example/tree/kratos_v1_1_0_selfservice)

手順2.は、Recovery flowを更新する際のパラメーターにEmailを指定するだけで、Registration flowの更新と同様のため、コード解説は割愛します。

手順3.では、メールに送信されたアカウント復旧コードを指定して、Recovery flowを更新すると、パスワード変更のためのSettings flowが作成され、またログインセッションも作成されます。（`Browser-based flows`の場合はCookieでログインセッションが返却されます）

### Recovery flowの更新後のSettings flowへの遷移

Settings flowを使用することでパスワードの変更が可能ですが、Settings flowの実行には、ログイン状態である必要があります。

アカウント復旧（パスワードリセット）時は、通常のログインができない状態であるため、Recovery flowの実行によって取得したログインセッションを使用する必要があります。

Recovery flowの完了後には、Settings flowが作成されますが、Registration flow -> Verification flowの場合のように、そのまま継続するのではなく、一度ブラウザ上でリダイレクトが必要です。

CSRF等のセキュリティを確保するためだと思いますが、以下で関連トピックが議論されているようです。

https://github.com/ory/kratos/discussions/2959

https://github.com/ory/kratos/issues/2884

Recovery flowの更新後に成功すると、status code 422 (browser_location_change_required) のエラーが返却されるのですが、エラー内の`redirect_browser_to`にリダイレクト先のURLが含まれています。

```go:app/auth-general/kratos/selfservice.go 
func (p *Provider) UpdateRecoveryFlow(i UpdateRecoveryFlowInput) (UpdateRecoveryFlowOutput, error) {
  ...

	// Recovery Flow を更新
	updateRecoveryFlowBody := kratosclientgo.UpdateRecoveryFlowBody{
		UpdateRecoveryFlowWithCodeMethod: &updateBody,
	}
	recoveryFlow, response, err := p.kratosPublicClient.FrontendApi.
		UpdateRecoveryFlow(context.Background()).
		Flow(i.FlowID).
		Cookie(i.Cookie).
		UpdateRecoveryFlowBody(updateRecoveryFlowBody).
		Execute()
	if err != nil {
		slog.Error("Update Recovery Flow Error", "RecoveryFlow", recoveryFlow, "Response", response, "Error", err)
		// browser location changeが返却された場合は、リダイレクト先URLを設定
		if response.StatusCode == 422 {
			output.RedirectBrowserTo = getRedirectBrowserToFromError(err)
			output.Cookies = response.Header["Set-Cookie"]
		} else {
			output.ErrorMessages = getErrorMessages(err)
		}
		return output, err
	}
	slog.Info("UpdateRecovery Succeed", "RecoveryFlow", recoveryFlow, "Response", response)

	// browser flowでは、kartosから受け取ったcookieをそのままブラウザへ返却する
	output.Cookies = response.Header["Set-Cookie"]

	return output, nil
}
```

### Seggins flowの更新 (state -> success)

`redirect_browser_to`には、Settings flowのUI URLが`?flow=`のクエリパラメータ入りで入っています。

UI URLは、kratosのconfigで設定可能です。

```yaml:kratos/config.yml
selfservice:
  ...
  flows:
    settings:
      ui_url: http://localhost:3000/my/password
			...
```

上記のUI URLへにリダイレクトされた際に、クエリパラメータで付与されたflow IDから、作成されたSettings flowを取得します。

flow IDが指定された場合は、CreateOrGetSettingsFlow関数で作成済みのSettings flowを取得しています。

(Settings flowには、更新できる対象として、password以外にもTraits内の項目(Emailや、例えばNickname等のプロフィール項目)の更新が可能です。)

```go:app/auth-general/handler/handler_auth.go
type handleGetMyPasswordRequestParams struct {
	cookie string
	flowID string
}

func (p *Provider) handleGetMyPassword(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	session := getSession(ctx)

	reqParams := handleGetMyPasswordRequestParams{
		cookie: r.Header.Get("Cookie"),
		flowID: r.URL.Query().Get("flow"),
	}

	// Setting Flow の作成 or 取得
	// Setting flowを新規作成した場合は、FlowIDを含めてリダイレクト
	output, err := p.d.Kratos.CreateOrGetSettingsFlow(kratos.CreateOrGetSettingsFlowInput{
		Cookie: reqParams.cookie,
		FlowID: reqParams.flowID,
	})

	if err != nil {
		tmpl.ExecuteTemplate(w, "my/password/index.html", viewParameters(session, r, map[string]any{
			"ErrorMessages": output.ErrorMessages,
		}))
	}

	// kratosのcookieをそのままブラウザへ受け渡す
	setCookieToResponseHeader(w, output.Cookies)

	// flowの情報に従ってレンダリング
	tmpl.ExecuteTemplate(w, "my/password/index.html", viewParameters(session, r, map[string]any{
		"SettingsFlowID":       output.FlowID,
		"CsrfToken":            output.CsrfToken,
		"RedirectFromRecovery": reqParams.flowID == "recovery",
	}))
}
```

```html:app/sample/templates/my/password/_form.html
<form 
  id="password-form"
  hx-post="/my/password?flow={{.SettingsFlowID}}" 
  hx-swap="outerHTML" 
  hx-target="this"
>
  <input
    name="csrf_token"
    type="hidden"
    value="{{.CsrfToken}}"
  />
  <div class="mt-2 mb-4">
    <label class="form-control">
      <div class="label">
        <span class="label-text">パスワード</span>
      </div>
      <input 
        id="password"
        type="password" 
        name="password" 
        value="Overwatch2024!@"
        class="input input-bordered"
        onkeyup="this.setCustomValidity('')"
        hx-on:htmx:validation:validate="
          if(this.value != document.getElementById('password-confirmation').value) {
            this.setCustomValidity('パスワードが一致しません') 
            htmx.find('#settings-form').reportValidity()
          }
        "
      >
    </label>

    <label class="form-control">
      <div class="label">
        <span class="label-text">パスワード確認</span>
      </div>
      <input 
        id="password-confirmation"
        type="password" 
        name="password-confirmation" 
        value="Overwatch2024!@"
        class="input input-bordered"
        onkeyup="this.setCustomValidity('')"
      >
    </label>
  </div>

  <div class="mx-auto text-center">
    <button class="btn btn-primary btn-wide">送信</button>
  </div>
```

上記画面から、`POST /my/password`へAJAXでアクセスし、正常にflowを更新できれば完了です。

もしエラーがあればレスポンスとしてHTMLフラグメントが返却されます。

### まとめ
ブラウザから、kratosの各種self-service flowを実行するサンプルコードを紹介しました。

kratosへのアクセスをAPIでラッピングし、レンダリングにHTMXを使用したHDAの構成でのサンプルです。

kratosが持つセキュリティの恩恵に預かるためには、kratosのself-service flowを理解した実装が必要です。

flowによっては、Registration flow から Verification flow へ遷移するなど、flow間の遷移についても理解が必要で、仕様理解が大変な部分だと思います。

本サンプルは、私自身の実装リファレンスとしての役割が主ですが、よろしければご参考にしてください。

