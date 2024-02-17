---
title: kratos browser flow example
emoji: ":lock" 
type: "tech" 
topics: ["ory", "kratos", "authentication", "htmx", "golang"] 
published: false
---  

## 本記事の概要
以下の記事で、ory kratosのユースケース概要と、shell scriptを使用した簡単な実行サンプルを紹介しました。

[ID管理 & 認証APIの ory kratos(self-hosting)の紹介](https://zenn.dev/yoshinori_satoh/articles/kartos_usecase_overview)

本記事では、ブラウザからkratosの各種self-service flowを実行するサンプルコードを紹介・解説します。

## self-service flow

### flowの初期化とレンダリング

### flowの実行

### flow間の遷移

#### Registration flow から Verification flow への遷移

#### Recovery flow から Settings flow への遷移



## サンプルの構成

### ディレクトリ構成

### kratosへのアクセスはAPI経由

### HTMXを使用したレンダリング

SPAではなく、HDA

self-service flowでは、server side appsの部分の実装に該当する


## Registration flow

### Registration flowの初期化とレンダリング

### Registration flowの実行

### Verfiication flowへの遷移

Registration flowが完了すると、Emailを検証するためのVerification flowが作成され、Verification flow IDが返却されます。

```go:app/auth-general/kratos/selfservice.go UpdateRegistrationFlow
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

CSRF Token取得と同様に、Verification flow ID取得の際も、kratos-client-goのSDKに不具合があるようなので、http.Responseから取得しています。

ここで取得したVerification flow IDを使用して、Verification flowの継続が必要です。

### Verfiication flow内部のステップと状態
Verification flowには以下のステップがあり、またflowの状態が`selfservice_verification_flows`テーブルの`state`カラムに保管されています。

[TODO: 図]

それぞれのステップに対応して、stateが更新されます。

1. Verification flowの作成 (state -> choose_method) 
2. 検証したいEmailを使用して、flowを更新 (state -> sent_email)
3. 2.で指定したEmailへ送信された、検証コードを使用して、flowを更新 (state -> passed_challenge)

Registration flowでは、flowを作成してそれを更新するという手順のみでしたが、Verification flowはもう少し複雑な手順です。

Registration flow完了後に、kratos側で上記の手順2.までが実行され、検証対象のEmailへ検証コードが送信されます。

Registration flowからVerification flowへ遷移する場合は、検証対象のEmailは、Registartion flowで登録に使用したEmailとなります。

残りは手順4.のみであり、登録したEmailへ送信された検証コードと、Registration flow完了時に返却されたVerification flow IDを使用して、Verification flowを更新します。

### Verification flowの更新 (state -> passed_challenge)

Registration flow完了時に返却されたVerification flow IDを使用して、検証コード入力フォームをレンダリングします。

![](https://github.com/YoshinoriSatoh/zenn/blob/master/images/kratos_browser_flow_example/verification_code_input.png?raw=true)

上記の画面をレンダリングするHTMLテンプレートです。

```html:app/auth-general/templates/verification/_code_form.html
{{define "auth/verification/_code_form.html"}}
<div class="alert alert-info mt-2">
  <div>
    <div>アカウント検証メールが送信されました。</div>
    <div>メールに記載されている6桁の検証コードを入力してください。 </div>
    <a class="link" href="http://localhost:4436" target="_blank">localhostのメールサーバはこちら</a>
  </div>
</div>
<form 
  id="verification-form" 
  hx-post="{{ .RoutePaths.AuthVerificationCode }}?flow={{.VerificationFlowID}}"
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
{{end}}
```

上記HTMLのレンダリングには、`VerificationFlowID`と`CsrfToken`が埋め込まれています。

flow ID から flow情報を取得可能なGET APIがkratosに用意されており、これを利用してflow情報からCSRF Tokenを取得しています。

上記HTMLのformが送信されると（HTMXを使用しているのでAJAXですが）、最終的に以下の関数が呼び出されます。

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

このため、Emailが指定された場合は、`2. 検証したいEmailを使用して、flowを更新 (state -> sent_email)`を、Codeが指定された場合は`3. 2.で指定したEmailへ送信された、検証コードを使用して、flowを更新 (state -> passed_challenge)`を実行します。

## Registration flow

#### 補足
Registration flowからVerification flowへ遷移する挙動は、Identity Schemaの IdentifierにEmailを指定し、なおかつEmailを使用してVerificationを実行するように指定している場合に限ります。

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

### おわりに
ブラウザから、kratosの各種self-service flowを実行するサンプルコードを紹介しました。

kratosへのアクセスをAPIでラッピングし、レンダリングにHTMXを使用したHDAの構成でのサンプルです。

kratosが持つセキュリティの恩恵に預かるためには、kratosのself-service flowを理解した実装が必要です。

flowによっては、Registration flow から Verification flow へ遷移するなど、flow間の遷移についても理解が必要で、仕様理解が大変な部分だと思います。

本サンプルは、私自身の実装リファレンスとしての役割が主ですが、よろしければご参考にしてください。

