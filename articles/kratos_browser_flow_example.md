---
title: 
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



ここで取得したVerification flow IDを使用して、Verification flowを継続する



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

