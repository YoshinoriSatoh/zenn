---
title: ory kratos APIの実行サンプル
emoji: "📦" 
type: "tech" 
topics: ["ory", "kratos", "authentication"] 
published: false
---  

## 本記事の概要

[こちら](https://zenn.dev/yoshinori_satoh/articles/kartos_usecase_overview)の記事のサンプルとして、[kratos self-service flowのAPI実行サンプルのリポジトリ](https://github.com/YoshinoriSatoh/kratos_selfservice_example)を作成しています。

このリポジトリのREADMEの内容ほぼそのままですが、kratos APIの使用方法理解の助けになるかと思い、zennにもアップします。

## サンプル概要
ローカルでkratos一式を起動するためのdocker composeと、Browser/APIそれぞれのSelfService flowをcurlで再現するbashスクリプトを用意しています。

Browser flowには、[サーバーサイドレンダリングの場合](https://www.ory.sh/docs/kratos/self-service#browser-flows-for-server-side-apps-nodejs-php-java-)と、[クライアントサイドレンダリングの場合](https://www.ory.sh/docs/kratos/self-service#browser-flows-for-client-side-apps-single-page-apps-reactjs-angular-nextjs-)がありますが、ここではクライアントサイドレンダリングの場合を想定しています。

## 構成
docker-compose.yaml で以下のコンテナが起動します。

| container | description |
| ---- | ---- |
| kratos | kratos本体です |
| kratos-migrate | kratos DBに対してマイグレーションを行います |
| db-kratos | kratosのDBです(PostgreSQL) |
| mailslurper | メールサーバーです |

### SelfService flow実行スクリプト
./scripts配下に各種SelfService flowを実行するスクリプトを格納しています。

| flow type | operation | script file |
| ---- | ---- | ---- |
| browser | registration | registration_browser.sh |
| browser | login | login_browser.sh |
| browser | check session | whoami_browser.sh |
| browser | verification | verification_browser.sh |
| browser | recovery | recovery_browser.sh |
| browser | settings (password) | settings_password_browser.sh |
| browser | settings (profile) | settings_profile_browser.sh |
| browser | logout | logout_browser.sh |
| api | registration | registration_api.sh |
| api | login | login_api.sh |
| api | check session | whoami_api.sh |
| api | verification | verification_api.sh |
| api | settings (password) | settings_password_api.sh |
| api | settings (profile) | settings_profile_api.sh |
| api | logout | logout_api.sh |

**注意点**

[API flowについて、recoverはサポートされていません](https://github.com/ory/kratos/discussions/2959)

recoveryはリダイレクトが必要となるため、ここだけはブラウザで実装する必要があるようです。

## 起動

### docker compose
```
docker compose up
```

## エンドポイントなど

| 項目 | URL |
| ---- | ---- |
| kratos public endpoint | http://localhost:4533 |
| kratos admin endpoint | http://localhost:4534 |
| kratos DB | postgres://kratos:secret@localhost:5432/kratos |
| mailslurper console | http://localhost:4436 |


## Identity schema

本サンプルでは以下のIdentity schemaを想定しています。

```json
{
  "$id": "https://schemas.ory.sh/presets/kratos/quickstart/email-password/identity.schema.json",
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "user",
  "type": "object",
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
      },
      "required": [
        "email"
      ],
      "additionalProperties": false
    }
  }
}
```

## browser flow(SPA)実行例

一旦、browser flowのみ記載します。

API flowはまたいずれ追記したいと思います。

### cookieの保管
curlによるブラウザの操作を再現しており、cookieを`.session_cookie`ファイルに保管しています。

### ユーザー登録
以下が実行されます。
1. Registration flow初期化API
2. Registration flow送信API(method: password)
3. 2.で実行されたVerification flowによるメールアドレス検証メール確認と検証コード入力
4. Verification flow取得API
5. Verification flow(mothod: code)送信API

#### コマンド実行手順
```
./scripts/registration_browser.sh <email> <password>
```

上記実行後に以下のプロンプトが表示されます。

```
please input code emailed to you:
```

[mailslurper console](http://localhost:4436)へアクセスすると、"Please verify your email address"というメールが届いています。

メール本文中に記載されている6桁の検証コードをプロンプトに入力し、Enterキーを押下すると、5. Verification flow(mothod: code)送信APIが実行され、メールアドレスが検証された状態となります。

#### 実行例
```
./scripts/registration_browser.sh 1@local overwatch2023
```

#### 1. Registration flowの初期化API

endpoint: `GET {{ kratos public endpoint }}/self-service/registration/browser`

[APIドキュメント](https://www.ory.sh/docs/kratos/reference/api#tag/frontend/operation/createBrowserRegistrationFlow)

Registration flowの初期化を行います。

レスポンスには、flow id等の他、uiという項目が含まれています。

uiで返却された項目は、本来はUIのレンダリングに使用します。

[ドキュメントによると](https://www.ory.sh/docs/kratos/self-service#form-rendering-1)、SPAの場合も[サーバサイドレンダリングの場合と同様に](https://www.ory.sh/docs/kratos/self-service#form-rendering)レンダリングする必要があるとのことです。

但し、本サンプルではcurlを使用していますので、UIレンダリングの過程は省いています。

**フォームレンダリング例**
```html
<form action="http://localhost:4533/self-service/registration?flow=63ef44ef-1c95-4792-9bf5-0fd0b3def246" method="POST">
  <input
    name="csrf_token"
    type="hidden"
    value="6SjBGe5CvyelZfs9lbscfPsqvfWDWUxAm7mrZwOmGv75eeUJ8v3VZbpIH16mrJRjJzl42+qIMxxidMpuoxnc3g=="
  />
  <fieldset>
    <label>
      <input name="traits.email" type="email" value="" placeholder="E-Mail" />
      <span>E-Mail</span>
    </label>
  </fieldset>
  <fieldset>
    <label>
      <input name="password" type="password" value="" placeholder="Password" />
      <span>Password</span>
    </label>
  </fieldset>
  <fieldset>
    <label>
      <input name="traits.nickname" type="text" value="" placeholder="nickname" />
      <span>nickname</span>
    </label>
  </fieldset>
  <fieldset>
    <label>
      <input name="traits.birthdat" type="text" value="" placeholder="birthdate" />
      <span>birthdate</span>
    </label>
  </fieldset> 
  <button name="method" type="submit" value="password">Sign up</button>
</form>
```


**レスポンス例**
```json
{
  "id": "63ef44ef-1c95-4792-9bf5-0fd0b3def246",
  "type": "browser",
  "expires_at": "2024-01-31T10:47:47.927093211Z",
  "issued_at": "2024-01-31T09:47:47.927093211Z",
  "request_url": "http://localhost:4533/self-service/registration/browser",
  "ui": {
    "action": "http://localhost:4533/self-service/registration?flow=63ef44ef-1c95-4792-9bf5-0fd0b3def246",
    "method": "POST",
    "nodes": [
      {
        "type": "input",
        "group": "default",
        "attributes": {
          "name": "csrf_token",
          "type": "hidden",
          "value": "6SjBGe5CvyelZfs9lbscfPsqvfWDWUxAm7mrZwOmGv75eeUJ8v3VZbpIH16mrJRjJzl42+qIMxxidMpuoxnc3g==",
          "required": true,
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {}
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "traits.email",
          "type": "email",
          "required": true,
          "autocomplete": "email",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "E-Mail",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "password",
          "type": "password",
          "required": true,
          "autocomplete": "new-password",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070001,
            "text": "Password",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "traits.nickname",
          "type": "text",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "nickname",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "traits.birthdate",
          "type": "text",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "birthdate",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "method",
          "type": "submit",
          "value": "password",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1040001,
            "text": "Sign up",
            "type": "info",
            "context": {}
          }
        }
      }
    ]
  }
}
```

#### 2. Registration flowの実行API(method: password)

endpoint: `POST {{ kratos public endpoint }}/self-service/registration/browser`

[APIドキュメント](https://www.ory.sh/docs/kratos/reference/api#tag/frontend/operation/updateRegistrationFlow)

1.で初期化したRegistration flowを実行します。

ここでは、認証にpasswordを使用しています。

(他には、oidcやwebauthnが指定可能です。)

1.のレスポンスを参照してレンダリングされたinput情報と、cookieが付与されている必要があります。

(curlの場合は明示的にcookieを付与していますが、ブラウザの場合は意識することはありません。)

レスポンスの`continue_with.flow.url`にリダイレクト先のURLが含まれています。

Identity schemaで、emailをcredentialsに指定している場合、Registration flowの実行API(method: password)を実行時に、メールアドレスを検証するためのVerification flowが実行されます。

Registration flowからVerification flowへ切り替わるため、次のflowを継続するための情報が`continue_with`に含まれています。

2024年1月現在、ドキュメントに明確な記載はないようなのですが、UI側で`continue_with.flow.url`へリダイレクトし、クエリパラメータで指定されたflow idから、以下のVerification flow取得APIを呼び出して、Verification flowを実行してほしいという意図があるのではないかと思います。

（本サンプルでは、curlを使用しているため、レンダリングの過程は省いています。）

**レスポンス例**
```json
{
  "session": {
    "id": "11e5917c-9ddd-4139-8e82-60d9ea3a6b06",
    "active": true,
    "expires_at": "2024-02-01T09:47:48.424595294Z",
    "authenticated_at": "2024-01-31T09:47:48.431011544Z",
    "authenticator_assurance_level": "aal1",
    "authentication_methods": [
      {
        "method": "password",
        "aal": "aal1",
        "completed_at": "2024-01-31T09:47:48.424594836Z"
      }
    ],
    "issued_at": "2024-01-31T09:47:48.424595294Z",
    "identity": {
      "id": "584cd5aa-fe7a-4b62-adfb-8452e4dd33bc",
      "schema_id": "user_v1",
      "schema_url": "http://localhost:4533/schemas/dXNlcl92MQ",
      "state": "active",
      "state_changed_at": "2024-01-31T09:47:48.418015253Z",
      "traits": {
        "email": "1@local"
      },
      "verifiable_addresses": [
        {
          "id": "a8700929-bedf-4d90-a981-dbdfc01ff75f",
          "value": "1@local",
          "verified": false,
          "via": "email",
          "status": "sent",
          "created_at": "2024-01-31T09:47:48.420204Z",
          "updated_at": "2024-01-31T09:47:48.420204Z"
        }
      ],
      "recovery_addresses": [
        {
          "id": "02924cde-27fb-4b32-9ec5-e025421d8312",
          "value": "1@local",
          "via": "email",
          "created_at": "2024-01-31T09:47:48.421036Z",
          "updated_at": "2024-01-31T09:47:48.421036Z"
        }
      ],
      "metadata_public": null,
      "created_at": "2024-01-31T09:47:48.419256Z",
      "updated_at": "2024-01-31T09:47:48.419256Z"
    },
    "devices": [
      {
        "id": "bb51ec7c-79e3-4c02-808b-758f62c1275c",
        "ip_address": "192.168.65.1:39001",
        "user_agent": "curl/7.87.0",
        "location": ""
      }
    ]
  },
  "identity": {
    "id": "584cd5aa-fe7a-4b62-adfb-8452e4dd33bc",
    "schema_id": "user_v1",
    "schema_url": "http://localhost:4533/schemas/dXNlcl92MQ",
    "state": "active",
    "state_changed_at": "2024-01-31T09:47:48.418015253Z",
    "traits": {
      "email": "1@local"
    },
    "verifiable_addresses": [
      {
        "id": "a8700929-bedf-4d90-a981-dbdfc01ff75f",
        "value": "1@local",
        "verified": false,
        "via": "email",
        "status": "sent",
        "created_at": "2024-01-31T09:47:48.420204Z",
        "updated_at": "2024-01-31T09:47:48.420204Z"
      }
    ],
    "recovery_addresses": [
      {
        "id": "02924cde-27fb-4b32-9ec5-e025421d8312",
        "value": "1@local",
        "via": "email",
        "created_at": "2024-01-31T09:47:48.421036Z",
        "updated_at": "2024-01-31T09:47:48.421036Z"
      }
    ],
    "metadata_public": null,
    "created_at": "2024-01-31T09:47:48.419256Z",
    "updated_at": "2024-01-31T09:47:48.419256Z"
  },
  "continue_with": [
    {
      "action": "show_verification_ui",
      "flow": {
        "id": "f13fa3be-b40b-4ed7-9bd8-57cf641785d5",
        "verifiable_address": "1@local",
        "url": "http://localhost:8000/auth/verification?flow=f13fa3be-b40b-4ed7-9bd8-57cf641785d5"
      }
    }
  ]
}
```

#### 3. 2.で実行されたVerification flowによるメールアドレス検証メール確認と検証コード入力
2.で実行されたVerification flowによって、メールアドレス検証用のメールアドレスが送信されています。

メール本文中には6桁の検証コードが記載されており、[mailslurper console](http://localhost:4436)へアクセスすることで、ローカルで受信メールを確認できます。

**メールアドレス検証メール例**
```
Hi, please verify your account by entering the following code: 312996 or clicking the following link: http://localhost:4533/self-service/verification?code=312996&flow=d229d11d-8273-4b7e-b05e-57490c0310f0
```

メール本文中に記載されている6桁の検証コードを以下のプロンプトに入力し、Enterキーを押下すると、4. Verification flow(mothod: code)送信APIが実行されます。

```
please input code emailed to you:
```

#### 4. Verification flow取得API
endpoint: `GET {{ kratos public endpoint }}/self-service/verification/flows`

[APIドキュメント](https://www.ory.sh/docs/kratos/reference/api#tag/frontend/operation/getVerificationFlow)

改めてVerification flowを取得しています。

ここで取得したcsrf_tokenが6. Settings flow(mothod: password)送信APIのリクエストボディに必要となります。

[Registration flowと同様に](https://github.com/YoshinoriSatoh/kratos_selfservice_example?tab=readme-ov-file#1-registration-flow%E3%81%AE%E5%88%9D%E6%9C%9F%E5%8C%96api)、uiの内容に従ってUIをレンダリングします。

**レスポンス例**
```json
{
  "id": "f13fa3be-b40b-4ed7-9bd8-57cf641785d5",
  "type": "browser",
  "expires_at": "2024-01-31T10:47:48.426066Z",
  "issued_at": "2024-01-31T09:47:48.426066Z",
  "request_url": "http://localhost:4533/self-service/registration/browser?return_to=",
  "active": "code",
  "ui": {
    "action": "http://localhost:4533/self-service/verification?flow=f13fa3be-b40b-4ed7-9bd8-57cf641785d5",
    "method": "POST",
    "nodes": [
      {
        "type": "input",
        "group": "code",
        "attributes": {
          "name": "code",
          "type": "text",
          "required": true,
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070011,
            "text": "Verification code",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "code",
        "attributes": {
          "name": "method",
          "type": "hidden",
          "value": "code",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {}
      },
      {
        "type": "input",
        "group": "code",
        "attributes": {
          "name": "method",
          "type": "submit",
          "value": "code",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070005,
            "text": "Submit",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "default",
        "attributes": {
          "name": "csrf_token",
          "type": "hidden",
          "value": "MVGaeJEn5vJDOVPLDY1lxTs+sdPTIKDnqMKZ04jeL01YHlHywiYDPH1DmGUFTHVV2rJjzDu333F9z0m9BCXPWg==",
          "required": true,
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {}
      }
    ],
    "messages": [
      {
        "id": 1080003,
        "text": "An email containing a verification code has been sent to the email address you provided. If you have not received an email, check the spelling of the address and make sure to use the address you registered with.",
        "type": "info",
        "context": {}
      }
    ]
  },
  "state": "sent_email"
}
```

#### 5. Verification flow(mothod: code)送信API

endpoint: `POST {{ kratos public endpoint }}/self-service/verification`

[APIドキュメント](https://www.ory.sh/docs/kratos/reference/api#tag/frontend/operation/updateVerificationFlow)

Verification flow(mothod: code)送信APIが呼び出し、メールアドレスが検証された状態となります。

kratosコンフィグで以下の設定をしているため、メールアドレスの検証が完了していないと、ログインできないようになっています。

```yaml
selfservice:
  flows:
    login:
      after:
        hooks:
          - hook: require_verified_address
```

```json
{
  "id": "f13fa3be-b40b-4ed7-9bd8-57cf641785d5",
  "type": "browser",
  "expires_at": "2024-01-31T10:47:48.426066Z",
  "issued_at": "2024-01-31T09:47:48.426066Z",
  "request_url": "http://localhost:4533/self-service/registration/browser?return_to=",
  "active": "code",
  "ui": {
    "action": "http://localhost:8000/auth",
    "method": "GET",
    "nodes": [
      {
        "type": "input",
        "group": "default",
        "attributes": {
          "name": "csrf_token",
          "type": "hidden",
          "value": "oDv2URL+6ebh+F1R/iNg+hnPMg4XOGU1jPsM7i7uq9rJdD3bQf8MKN+Clv/24nBq+EPgEf+vGqNZ9tyAohVLzQ==",
          "required": true,
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {}
      },
      {
        "type": "a",
        "group": "code",
        "attributes": {
          "href": "http://localhost:8000/auth",
          "title": {
            "id": 1070009,
            "text": "Continue",
            "type": "info"
          },
          "id": "continue",
          "node_type": "a"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070009,
            "text": "Continue",
            "type": "info"
          }
        }
      }
    ],
    "messages": [
      {
        "id": 1080002,
        "text": "You successfully verified your email address.",
        "type": "success"
      }
    ]
  },
  "state": "passed_challenge"
}
```

### ログイン
以下が実行されます。
1. Login flow初期化API
2. Login flow送信API

#### コマンド実行手順
```
./scripts/login_browser.sh <email> <password>
```

#### 実行例
```
./scripts/login_browser.sh 1@local overwatch2023
```

#### 1. Login flowの初期化API

endpoint: `GET {{ kratos public endpoint }}/self-service/login/browser`

[APIドキュメント](https://www.ory.sh/docs/kratos/reference/api#tag/frontend/operation/createBrowserLoginFlow)

Login flowの初期化を行います。

[Registration flowと同様に](https://github.com/YoshinoriSatoh/kratos_selfservice_example?tab=readme-ov-file#1-registration-flow%E3%81%AE%E5%88%9D%E6%9C%9F%E5%8C%96api)、uiの内容に従ってUIをレンダリングします。

**レスポンス例**
```json
{
  "id": "85a83b3d-835a-4ef1-a2e3-2e7d7cf8f826",
  "type": "browser",
  "expires_at": "2024-01-31T05:08:59.442571343Z",
  "issued_at": "2024-01-31T04:08:59.442571343Z",
  "request_url": "http://localhost:4533/self-service/login/browser",
  "ui": {
    "action": "http://localhost:4533/self-service/login?flow=85a83b3d-835a-4ef1-a2e3-2e7d7cf8f826",
    "method": "POST",
    "nodes": [
      {
        "type": "input",
        "group": "default",
        "attributes": {
          "name": "csrf_token",
          "type": "hidden",
          "value": "Dx5gyPsVHX1U2OYriH8qYxLT/6P8G5TuyROFqhyNGUQI4zMPaE0vrTC3jFmNBYHuofb9KUgN4/bzdc9yunIpXg==",
          "required": true,
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {}
      },
      {
        "type": "input",
        "group": "default",
        "attributes": {
          "name": "identifier",
          "type": "text",
          "value": "",
          "required": true,
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070004,
            "text": "ID",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "password",
          "type": "password",
          "required": true,
          "autocomplete": "current-password",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070001,
            "text": "Password",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "method",
          "type": "submit",
          "value": "password",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1010001,
            "text": "Sign in",
            "type": "info",
            "context": {}
          }
        }
      }
    ]
  },
  "created_at": "2024-01-31T04:08:59.446151Z",
  "updated_at": "2024-01-31T04:08:59.446151Z",
  "refresh": false,
  "requested_aal": "aal1"
}
```

#### 2. Login flowの送信API

1.で初期化したLogin flowを実行します。

session情報が返却されます。

```json
{
  "session": {
    "id": "0f867e3b-7c89-432c-9368-f021f4f686d4",
    "active": true,
    "expires_at": "2024-02-01T04:08:59.829461593Z",
    "authenticated_at": "2024-01-31T04:08:59.829461593Z",
    "authenticator_assurance_level": "aal1",
    "authentication_methods": [
      {
        "method": "password",
        "aal": "aal1",
        "completed_at": "2024-01-31T04:08:59.829459176Z"
      }
    ],
    "issued_at": "2024-01-31T04:08:59.829461593Z",
    "identity": {
      "id": "793126a9-3c8b-43ec-89d0-e48395235131",
      "schema_id": "user_v1",
      "schema_url": "http://localhost:4533/schemas/dXNlcl92MQ",
      "state": "active",
      "state_changed_at": "2024-01-31T03:53:49.789625Z",
      "traits": {
        "email": "1@local"
      },
      "verifiable_addresses": [
        {
          "id": "a7d3f207-0a8d-47af-b0fb-576806a1bcde",
          "value": "1@local",
          "verified": true,
          "via": "email",
          "status": "completed",
          "verified_at": "2024-01-31T04:08:28.273878Z",
          "created_at": "2024-01-31T03:53:49.7915Z",
          "updated_at": "2024-01-31T03:53:49.7915Z"
        }
      ],
      "recovery_addresses": [
        {
          "id": "694551fc-4074-4b92-b8e2-8cfe0a67c2e6",
          "value": "1@local",
          "via": "email",
          "created_at": "2024-01-31T03:53:49.792294Z",
          "updated_at": "2024-01-31T03:53:49.792294Z"
        }
      ],
      "metadata_public": null,
      "created_at": "2024-01-31T03:53:49.790597Z",
      "updated_at": "2024-01-31T03:53:49.790597Z"
    },
    "devices": [
      {
        "id": "98e9b8cc-5ffb-446c-b76d-7383b112600d",
        "ip_address": "192.168.65.1:38548",
        "user_agent": "curl/7.87.0",
        "location": ""
      }
    ]
  }
}
```




### ログインセッション取得
以下が実行されます。
1. Login session取得API

#### コマンド実行手順
```
./scripts/whoami_browser.sh <email> <password>
```

#### 実行例
```
./scripts/whoami_browser.sh
```

#### 1. Login session取得API

endpoint: `GET {{ kratos public endpoint }}/sessions/whoami`

[APIドキュメント](https://www.ory.sh/docs/kratos/reference/api#tag/frontend/operation/toSession)

ログイン中のセッションが有効であれば、セッション情報が返却されます。

セッション情報取得の他、現在ログイン中であるかどうかを確認するエンドポイントでもあります。

**レスポンス例**
```json
{
  "id": "0f867e3b-7c89-432c-9368-f021f4f686d4",
  "active": true,
  "expires_at": "2024-02-01T04:08:59.829461Z",
  "authenticated_at": "2024-01-31T04:08:59.829461Z",
  "authenticator_assurance_level": "aal1",
  "authentication_methods": [
    {
      "method": "password",
      "aal": "aal1",
      "completed_at": "2024-01-31T04:08:59.829459176Z"
    }
  ],
  "issued_at": "2024-01-31T04:08:59.829461Z",
  "identity": {
    "id": "793126a9-3c8b-43ec-89d0-e48395235131",
    "schema_id": "user_v1",
    "schema_url": "http://localhost:4533/schemas/dXNlcl92MQ",
    "state": "active",
    "state_changed_at": "2024-01-31T03:53:49.789625Z",
    "traits": {
      "email": "1@local"
    },
    "verifiable_addresses": [
      {
        "id": "a7d3f207-0a8d-47af-b0fb-576806a1bcde",
        "value": "1@local",
        "verified": true,
        "via": "email",
        "status": "completed",
        "verified_at": "2024-01-31T04:08:28.273878Z",
        "created_at": "2024-01-31T03:53:49.7915Z",
        "updated_at": "2024-01-31T03:53:49.7915Z"
      }
    ],
    "recovery_addresses": [
      {
        "id": "694551fc-4074-4b92-b8e2-8cfe0a67c2e6",
        "value": "1@local",
        "via": "email",
        "created_at": "2024-01-31T03:53:49.792294Z",
        "updated_at": "2024-01-31T03:53:49.792294Z"
      }
    ],
    "metadata_public": null,
    "created_at": "2024-01-31T03:53:49.790597Z",
    "updated_at": "2024-01-31T03:53:49.790597Z"
  },
  "devices": [
    {
      "id": "98e9b8cc-5ffb-446c-b76d-7383b112600d",
      "ip_address": "192.168.65.1:38548",
      "user_agent": "curl/7.87.0",
      "location": ""
    }
  ]
}
```


### アカウント復旧 (パスワードリセット)
以下が実行されます。
1. Recovery flow初期化API
2. Recovery flow送信API(method: code, send recovery email)
3. 2.で送信されたアカウント復旧メール確認とリカバリーコード入力 
4. Recovery flow送信API(method: code, send recovery code)
5. Settings flow取得API
6. Settings flow(mothod: password)送信API


本フローは少し複雑であるため、補足します。

([アカウント復旧に関するドキュメントはこちら](https://www.ory.sh/docs/kratos/self-service/flows/account-recovery-password-reset))

まず、他のflowと同様に、Recovery flowを初期化します。

次に、Recovery flowを実行するのですが、Recovery flowの中にも2段階のステップがあります。

Recovery flow(method: code)実行のステップ
* アカウント復旧メール送信
* アカウント復旧メール内のリカバリーコードを送信し、Settings flowを開始

初期化されたflowに対して、emailをリクエストボディに指定してflowを実行すると、アカウント復旧メールが送信されます。

届いたメール内のリカバリーコードをリクエストボディに指定して、もう一度本APIを実行します。

そうすると、パスワードを再設定可能なSettings flowが初期化されます。

ここで、Settings flowを実行するためのリダイレクト先URLが返却されます。

URLにはflow idがクエリパラメータに含まれているため、リダイレクト先で改めてSettings flowを取得します。

その後、再設定したいパスワードをリクエストボディに指定して、Settings flowを実行すると、パスワードが再設定されます。

上記で初期化されるSettings flowは、[特権セッション](https://www.ory.sh/docs/kratos/session-management/session-lifespan#privileged-sessions)が発行され、特権セッション期限内のみパスワードを再設定可能です。

#### コマンド実行手順
```
./scripts/recovery_browser.sh <email> <password>
```

上記実行後に以下のプロンプトが表示されます。

```
please input code emailed to you:
```

[mailslurper console](http://localhost:4436)へアクセスすると、"Recover access to your account"というメールが届いています。

メール本文中に記載されている6桁のアカウント復旧コードをプロンプトに入力し、Enterキーを押下すると、4. Recovery flow送信API(method: code, send recovery code)が実行され、アカウントのパスワードが再設定されます。

#### 実行例
```
./scripts/recovery_browser.sh 1@local overwatch2024
```

#### 1. Recovery flow初期化API

endpoint: `GET {{ kratos public endpoint }}/self-service/recovery/browser`

[APIドキュメント](https://www.ory.sh/docs/kratos/reference/api#tag/frontend/operation/createBrowserRecoveryFlow)

Recovery flowの初期化を行います。

[Registration flowと同様に](https://github.com/YoshinoriSatoh/kratos_selfservice_example?tab=readme-ov-file#1-registration-flow%E3%81%AE%E5%88%9D%E6%9C%9F%E5%8C%96api)、uiの内容に従ってUIをレンダリングします。

**レスポンス例**
```json
{
  "id": "3a6935f7-4b0b-4060-b770-50f8150040b7",
  "type": "browser",
  "expires_at": "2024-01-31T05:26:56.129178508Z",
  "issued_at": "2024-01-31T04:26:56.129178508Z",
  "request_url": "http://localhost:4533/self-service/recovery/browser",
  "active": "code",
  "ui": {
    "action": "http://localhost:4533/self-service/recovery?flow=3a6935f7-4b0b-4060-b770-50f8150040b7",
    "method": "POST",
    "nodes": [
      {
        "type": "input",
        "group": "default",
        "attributes": {
          "name": "csrf_token",
          "type": "hidden",
          "value": "rI0dFlZZ0adoMQMr7UpNKOLmzF6Y8y43ImL+fVKDD9agq2EPUkHYtG0aU/5wqorJoNFnlDKCs7sY0EC2k2xzog==",
          "required": true,
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {}
      },
      {
        "type": "input",
        "group": "code",
        "attributes": {
          "name": "email",
          "type": "email",
          "required": true,
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070007,
            "text": "Email",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "code",
        "attributes": {
          "name": "method",
          "type": "submit",
          "value": "code",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070005,
            "text": "Submit",
            "type": "info"
          }
        }
      }
    ]
  },
  "state": "choose_method"
}
```


#### 2. Recovery flow送信API(method: code, send recovery email)

endpoint: `POST {{ kratos public endpoint }}/self-service/recovery`

[APIドキュメント](https://www.ory.sh/docs/kratos/reference/api#tag/frontend/operation/updateRecoveryFlow)

1.で初期化したRecovery flowを実行します。

本章の冒頭で補足した中で「アカウント復旧メール送信」を実行するプロセスです。

リクエストボディにemailを指定して実行することで、アカウント復旧メールが送信されます。

**レスポンス例**
```json
{
  "id": "3a6935f7-4b0b-4060-b770-50f8150040b7",
  "type": "browser",
  "expires_at": "2024-01-31T05:26:56.129178Z",
  "issued_at": "2024-01-31T04:26:56.129178Z",
  "request_url": "http://localhost:4533/self-service/recovery/browser",
  "active": "code",
  "ui": {
    "action": "http://localhost:4533/self-service/recovery?flow=3a6935f7-4b0b-4060-b770-50f8150040b7",
    "method": "POST",
    "nodes": [
      {
        "type": "input",
        "group": "default",
        "attributes": {
          "name": "csrf_token",
          "type": "hidden",
          "value": "TQscVztoJW5otkXOHt5uYvPDk8GL0l5qNISI5rkSWxtBLWBOP3AsfW2dFRuDPqmDsfQ4CyGjw+YONjYteP0nbw==",
          "required": true,
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {}
      },
      {
        "type": "input",
        "group": "code",
        "attributes": {
          "name": "code",
          "type": "text",
          "required": true,
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070010,
            "text": "Recovery code",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "code",
        "attributes": {
          "name": "method",
          "type": "hidden",
          "value": "code",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {}
      },
      {
        "type": "input",
        "group": "code",
        "attributes": {
          "name": "method",
          "type": "submit",
          "value": "code",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070005,
            "text": "Submit",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "code",
        "attributes": {
          "name": "email",
          "type": "submit",
          "value": "1@local",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070008,
            "text": "Resend code",
            "type": "info"
          }
        }
      }
    ],
    "messages": [
      {
        "id": 1060003,
        "text": "An email containing a recovery code has been sent to the email address you provided. If you have not received an email, check the spelling of the address and make sure to use the address you registered with.",
        "type": "info",
        "context": {}
      }
    ]
  },
  "state": "sent_email"
}
```

#### 3. 2.で送信されたアカウント復旧メール確認とリカバリーコード入力 
2.で実行されたRecovery flowによって、アカウント復旧用のメールアドレスが送信されています。

メール本文中には6桁の検証コードが記載されており、[mailslurper console](http://localhost:4436)へアクセスすることで、ローカルで受信メールを確認できます。

**アカウント復旧メール例**
```
Hi, please recover access to your account by entering the following code: 653883
```

メール本文中に記載されている6桁のリカバリーコードを以下のプロンプトに入力し、Enterキーを押下すると、4. Recovery flow送信API(method: code, send recovery code)送信APIが実行されます。

```
please input code emailed to you:
```

#### 4. Recovery flow送信API(method: code, send recovery code)
endpoint: `POST {{ kratos public endpoint }}/self-service/recovery`

[APIドキュメント](https://www.ory.sh/docs/kratos/reference/api#tag/frontend/operation/updateRecoveryFlow)

Recovery flow(mothod: code)送信APIが呼び出し、アカウントのパスワード再設定が可能なSettings flowが初期化されます。

ここで、レスポンスにはエラーが返却されます。

**レスポンス例**
```json
{
  "error": {
    "id": "browser_location_change_required",
    "code": 422,
    "status": "Unprocessable Entity",
    "reason": "In order to complete this flow please redirect the browser to: http://localhost:8000/auth/settings?flow=4d1e39fa-2554-4a86-913a-b4ad2f36719a",
    "message": "browser location change required"
  },
  "redirect_browser_to": "http://localhost:8000/auth/settings?flow=4d1e39fa-2554-4a86-913a-b4ad2f36719a"
}
```

`browser_location_change_required`というエラーの通り、`redirect_browser_to`にリダイレクトをして、改めてSettings flowを継続する必要があります。

本サンプルでは、curlを使用しているため、リダイレクトは省いています。

#### 5. Settings flow取得API
endpoint: `GET {{ kratos public endpoint }}/self-service/settings/flows`

[APIドキュメント](https://www.ory.sh/docs/kratos/reference/api#tag/frontend/operation/getSettingsFlow)

改めてSettings flowを取得しています。

ここで取得したcsrf_tokenが6. Settings flow(mothod: password)送信APIのリクエストボディに必要となります。

[Registration flowと同様に](https://github.com/YoshinoriSatoh/kratos_selfservice_example?tab=readme-ov-file#1-registration-flow%E3%81%AE%E5%88%9D%E6%9C%9F%E5%8C%96api)、uiの内容に従ってUIをレンダリングします。

**レスポンス例**
```json
{
  "id": "4d1e39fa-2554-4a86-913a-b4ad2f36719a",
  "type": "browser",
  "expires_at": "2024-01-31T05:57:18.712901Z",
  "issued_at": "2024-01-31T04:57:18.712901Z",
  "request_url": "http://localhost:4533/self-service/recovery?flow=3a6935f7-4b0b-4060-b770-50f8150040b7",
  "ui": {
    "action": "http://localhost:4533/self-service/settings?flow=4d1e39fa-2554-4a86-913a-b4ad2f36719a",
    "method": "POST",
    "nodes": [
      {
        "type": "input",
        "group": "default",
        "attributes": {
          "name": "csrf_token",
          "type": "hidden",
          "value": "GO54xtABDHsxaOmCIJXvmtsJbO8ulYE4UL9bxklYw9MbFsNNiH3y7LkwvBwMBdysdnxpOHXGgf/4BJVn8x/TJQ==",
          "required": true,
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {}
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "traits.email",
          "type": "email",
          "value": "1@local",
          "required": true,
          "autocomplete": "email",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "E-Mail",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "traits.nickname",
          "type": "text",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "nickname",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "traits.birthdate",
          "type": "text",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "birthdate",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "method",
          "type": "submit",
          "value": "profile",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070003,
            "text": "Save",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "password",
          "type": "password",
          "required": true,
          "autocomplete": "new-password",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070001,
            "text": "Password",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "method",
          "type": "submit",
          "value": "password",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070003,
            "text": "Save",
            "type": "info"
          }
        }
      }
    ],
    "messages": [
      {
        "id": 1060001,
        "text": "You successfully recovered your account. Please change your password or set up an alternative login method (e.g. social sign in) within the next 60.00 minutes.",
        "type": "success",
        "context": {
          "privilegedSessionExpiresAt": "2024-01-31T05:57:18.720771463Z"
        }
      }
    ]
  },
  "identity": {
    "id": "793126a9-3c8b-43ec-89d0-e48395235131",
    "schema_id": "user_v1",
    "schema_url": "http://localhost:4533/schemas/dXNlcl92MQ",
    "state": "active",
    "state_changed_at": "2024-01-31T03:53:49.789625Z",
    "traits": {
      "email": "1@local"
    },
    "verifiable_addresses": [
      {
        "id": "a7d3f207-0a8d-47af-b0fb-576806a1bcde",
        "value": "1@local",
        "verified": true,
        "via": "email",
        "status": "completed",
        "verified_at": "2024-01-31T04:08:28.273878Z",
        "created_at": "2024-01-31T03:53:49.7915Z",
        "updated_at": "2024-01-31T03:53:49.7915Z"
      }
    ],
    "recovery_addresses": [
      {
        "id": "694551fc-4074-4b92-b8e2-8cfe0a67c2e6",
        "value": "1@local",
        "via": "email",
        "created_at": "2024-01-31T03:53:49.792294Z",
        "updated_at": "2024-01-31T03:53:49.792294Z"
      }
    ],
    "metadata_public": null,
    "created_at": "2024-01-31T03:53:49.790597Z",
    "updated_at": "2024-01-31T03:53:49.790597Z"
  },
  "state": "show_form"
}
```

#### 6. Settings flow(mothod: password)送信API
endpoint: `POST {{ kratos public endpoint }}/self-service/settingss`

[APIドキュメント](https://www.ory.sh/docs/kratos/reference/api#tag/frontend/operation/updateSettingsFlow)

Settings flowを実行します。

ここでは、期間限定の特権セッションでアクセスが許可されており、パスワードを変更可能です。

**レスポンス例**
```json
{
  "id": "4d1e39fa-2554-4a86-913a-b4ad2f36719a",
  "type": "browser",
  "expires_at": "2024-01-31T05:57:18.712901Z",
  "issued_at": "2024-01-31T04:57:18.712901Z",
  "request_url": "http://localhost:4533/self-service/recovery?flow=3a6935f7-4b0b-4060-b770-50f8150040b7",
  "ui": {
    "action": "http://localhost:4533/self-service/settings?flow=7f9d8a25-0265-4f53-9d39-3d0570668812",
    "method": "POST",
    "nodes": [
      {
        "type": "input",
        "group": "default",
        "attributes": {
          "name": "csrf_token",
          "type": "hidden",
          "value": "Z1NUvBScTRjtLtFwur3qUDjnFF+SQEHANnYfWdqLlcdkq+83TOCzj2V2hO6WLdlmlZIRiMkTQQeezdH4YMyFMQ==",
          "required": true,
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {}
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "traits.email",
          "type": "email",
          "value": "1@local",
          "required": true,
          "autocomplete": "email",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "E-Mail",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "traits.nickname",
          "type": "text",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "nickname",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "traits.birthdate",
          "type": "text",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "birthdate",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "method",
          "type": "submit",
          "value": "profile",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070003,
            "text": "Save",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "password",
          "type": "password",
          "required": true,
          "autocomplete": "new-password",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070001,
            "text": "Password",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "method",
          "type": "submit",
          "value": "password",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070003,
            "text": "Save",
            "type": "info"
          }
        }
      }
    ],
    "messages": [
      {
        "id": 1050001,
        "text": "Your changes have been saved!",
        "type": "success"
      }
    ]
  },
  "identity": {
    "id": "793126a9-3c8b-43ec-89d0-e48395235131",
    "schema_id": "user_v1",
    "schema_url": "http://localhost:4533/schemas/dXNlcl92MQ",
    "state": "active",
    "state_changed_at": "2024-01-31T03:53:49.789625Z",
    "traits": {
      "email": "1@local"
    },
    "verifiable_addresses": [
      {
        "id": "a7d3f207-0a8d-47af-b0fb-576806a1bcde",
        "value": "1@local",
        "verified": true,
        "via": "email",
        "status": "completed",
        "verified_at": "2024-01-31T04:08:28.273878Z",
        "created_at": "2024-01-31T03:53:49.7915Z",
        "updated_at": "2024-01-31T03:53:49.7915Z"
      }
    ],
    "recovery_addresses": [
      {
        "id": "694551fc-4074-4b92-b8e2-8cfe0a67c2e6",
        "value": "1@local",
        "via": "email",
        "created_at": "2024-01-31T03:53:49.792294Z",
        "updated_at": "2024-01-31T03:53:49.792294Z"
      }
    ],
    "metadata_public": null,
    "created_at": "2024-01-31T03:53:49.790597Z",
    "updated_at": "2024-01-31T03:53:49.790597Z"
  },
  "state": "success"
}
```

### セッティング (パスワード更新)
以下が実行されます。
1. Settings flow初期化API
2. Settings flow送信API (method: password)

ここでは、Settings flowの中でも、パスワード更新を実施します。

Settings flowでは、パスワードの他にも、ユーザー情報の更新、その他OIDC等の設定更新が可能です。

#### 前提条件
予めログインされたcookieが必要です。

(`.session_cookie`に保存されています。)

[こちらを参照して](https://github.com/YoshinoriSatoh/kratos_selfservice_example?tab=readme-ov-file#%E3%83%AD%E3%82%B0%E3%82%A4%E3%83%B3)ログインしてください。

#### コマンド実行手順
```
./scripts/settings_password_browser.sh <update-password>
```

#### 実行例
```
./scripts/settings_password_browser.sh updated-overwatch2024
```

#### 1. Settings flow初期化API

endpoint: `GET {{ kratos public endpoint }}/self-service/settings/browser`

[APIドキュメント](https://www.ory.sh/docs/kratos/reference/api#tag/frontend/operation/createBrowserSettingsFlow)

Settings flowの初期化を行います。

[Registration flowと同様に](https://github.com/YoshinoriSatoh/kratos_selfservice_example?tab=readme-ov-file#1-registration-flow%E3%81%AE%E5%88%9D%E6%9C%9F%E5%8C%96api)、uiの内容に従ってUIをレンダリングします。

**レスポンス例**
```json
{
  "id": "9f9803c5-95d9-46b6-93ca-7701ee187812",
  "type": "browser",
  "expires_at": "2024-01-31T09:54:11.474883208Z",
  "issued_at": "2024-01-31T08:54:11.474883208Z",
  "request_url": "http://localhost:4533/self-service/settings/browser",
  "ui": {
    "action": "http://localhost:4533/self-service/settings?flow=9f9803c5-95d9-46b6-93ca-7701ee187812",
    "method": "POST",
    "nodes": [
      {
        "type": "input",
        "group": "default",
        "attributes": {
          "name": "csrf_token",
          "type": "hidden",
          "value": "a9uFV7aKpnmAJQQk7RJ7THQepfD+fby1Yh2LjYReGXel3r5mDd3+UNboi3LiMoQ6bS/d8gVL/c6e4eSJ4uwjdg==",
          "required": true,
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {}
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "traits.email",
          "type": "email",
          "value": "1@local",
          "required": true,
          "autocomplete": "email",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "E-Mail",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "traits.nickname",
          "type": "text",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "nickname",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "traits.birthdate",
          "type": "text",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "birthdate",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "method",
          "type": "submit",
          "value": "profile",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070003,
            "text": "Save",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "password",
          "type": "password",
          "required": true,
          "autocomplete": "new-password",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070001,
            "text": "Password",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "method",
          "type": "submit",
          "value": "password",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070003,
            "text": "Save",
            "type": "info"
          }
        }
      }
    ]
  },
  "identity": {
    "id": "793126a9-3c8b-43ec-89d0-e48395235131",
    "schema_id": "user_v1",
    "schema_url": "http://localhost:4533/schemas/dXNlcl92MQ",
    "state": "active",
    "state_changed_at": "2024-01-31T03:53:49.789625Z",
    "traits": {
      "email": "1@local"
    },
    "verifiable_addresses": [
      {
        "id": "a7d3f207-0a8d-47af-b0fb-576806a1bcde",
        "value": "1@local",
        "verified": true,
        "via": "email",
        "status": "completed",
        "verified_at": "2024-01-31T04:08:28.273878Z",
        "created_at": "2024-01-31T03:53:49.7915Z",
        "updated_at": "2024-01-31T03:53:49.7915Z"
      }
    ],
    "recovery_addresses": [
      {
        "id": "694551fc-4074-4b92-b8e2-8cfe0a67c2e6",
        "value": "1@local",
        "via": "email",
        "created_at": "2024-01-31T03:53:49.792294Z",
        "updated_at": "2024-01-31T03:53:49.792294Z"
      }
    ],
    "metadata_public": null,
    "created_at": "2024-01-31T03:53:49.790597Z",
    "updated_at": "2024-01-31T03:53:49.790597Z"
  },
  "state": "show_form"
}
```

#### 2. Settings flow送信API (method: password)

endpoint: `POST {{ kratos public endpoint }}/self-service/settings`

[APIドキュメント](https://www.ory.sh/docs/kratos/reference/api#tag/frontend/operation/updateSettingsFlow)

1.で初期化したSettings flowを実行します。

ここでは、パスワードが更新されます。

**レスポンス例**
```json
{
  "id": "9f9803c5-95d9-46b6-93ca-7701ee187812",
  "type": "browser",
  "expires_at": "2024-01-31T09:54:11.474883Z",
  "issued_at": "2024-01-31T08:54:11.474883Z",
  "request_url": "http://localhost:4533/self-service/settings/browser",
  "ui": {
    "action": "http://localhost:4533/self-service/settings?flow=e54f2132-8c51-40db-9d9b-c52ded875083",
    "method": "POST",
    "nodes": [
      {
        "type": "input",
        "group": "default",
        "attributes": {
          "name": "csrf_token",
          "type": "hidden",
          "value": "fXL+AFxv2mF5AlIzNKn5XlTPxBrz2qx8G9jU3zZMXWazd8Ux5ziCSC/P3WU7iQYoTf68GAjs7QfnJLvbUP5nZw==",
          "required": true,
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {}
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "traits.email",
          "type": "email",
          "value": "1@local",
          "required": true,
          "autocomplete": "email",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "E-Mail",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "traits.nickname",
          "type": "text",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "nickname",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "traits.birthdate",
          "type": "text",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "birthdate",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "method",
          "type": "submit",
          "value": "profile",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070003,
            "text": "Save",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "password",
          "type": "password",
          "required": true,
          "autocomplete": "new-password",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070001,
            "text": "Password",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "method",
          "type": "submit",
          "value": "password",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070003,
            "text": "Save",
            "type": "info"
          }
        }
      }
    ],
    "messages": [
      {
        "id": 1050001,
        "text": "Your changes have been saved!",
        "type": "success"
      }
    ]
  },
  "identity": {
    "id": "793126a9-3c8b-43ec-89d0-e48395235131",
    "schema_id": "user_v1",
    "schema_url": "http://localhost:4533/schemas/dXNlcl92MQ",
    "state": "active",
    "state_changed_at": "2024-01-31T03:53:49.789625Z",
    "traits": {
      "email": "1@local"
    },
    "verifiable_addresses": [
      {
        "id": "a7d3f207-0a8d-47af-b0fb-576806a1bcde",
        "value": "1@local",
        "verified": true,
        "via": "email",
        "status": "completed",
        "verified_at": "2024-01-31T04:08:28.273878Z",
        "created_at": "2024-01-31T03:53:49.7915Z",
        "updated_at": "2024-01-31T03:53:49.7915Z"
      }
    ],
    "recovery_addresses": [
      {
        "id": "694551fc-4074-4b92-b8e2-8cfe0a67c2e6",
        "value": "1@local",
        "via": "email",
        "created_at": "2024-01-31T03:53:49.792294Z",
        "updated_at": "2024-01-31T03:53:49.792294Z"
      }
    ],
    "metadata_public": null,
    "created_at": "2024-01-31T03:53:49.790597Z",
    "updated_at": "2024-01-31T03:53:49.790597Z"
  },
  "state": "success"
}
```

### セッティング (プロファイル更新)
以下が実行されます。
1. Settings flow初期化API
2. Settings flow送信API (method: profile)
以下は`email`を更新した場合のみ実行されます。
3. 2.で実行されたVerification flowによるメールアドレス検証メール確認と検証コード入力
4. Verification flow取得API
5. Verification flow(mothod: code)送信API

ここでは、Settings flowの中でも、プロファイル更新を実施します。

プロファイル更新では、Identity schemaで定義された属性の更新が可能です。

`email`が`identifier`に指定されている項目が更新された場合（なおかつ、`verification: via: "email"`が設定されている場合）、メールアドレスは検証されていない状態となります。

別途、メールアドレスのVerification flowを実施する必要があります。

#### 前提条件
予めログインされたcookieが必要です。

(`.session_cookie`に保存されています。)

[こちらを参照して](https://github.com/YoshinoriSatoh/kratos_selfservice_example?tab=readme-ov-file#%E3%83%AD%E3%82%B0%E3%82%A4%E3%83%B3)ログインしてください。

#### コマンド実行手順
```
./scripts/settings_profile_browser.sh <<update-email> <update-nickname> <update-birthdate>>
```

#### 実行例
```
./scripts/settings_profile_browser.sh updated-1@local updated-nickname 2000-01-01
```

#### 1. Settings flow初期化API

endpoint: `GET {{ kratos public endpoint }}/self-service/settings/browser`

[APIドキュメント](https://www.ory.sh/docs/kratos/reference/api#tag/frontend/operation/createBrowserSettingsFlow)

Settings flowの初期化を行います。

[Registration flowと同様に](https://github.com/YoshinoriSatoh/kratos_selfservice_example?tab=readme-ov-file#1-registration-flow%E3%81%AE%E5%88%9D%E6%9C%9F%E5%8C%96api)、uiの内容に従ってUIをレンダリングします。

**レスポンス例**
```json
{
  "id": "38c9c3ee-45c8-4b98-9d8b-d4850f5e8408",
  "type": "browser",
  "expires_at": "2024-01-31T10:11:13.388073334Z",
  "issued_at": "2024-01-31T09:11:13.388073334Z",
  "request_url": "http://localhost:4533/self-service/settings/browser",
  "ui": {
    "action": "http://localhost:4533/self-service/settings?flow=38c9c3ee-45c8-4b98-9d8b-d4850f5e8408",
    "method": "POST",
    "nodes": [
      {
        "type": "input",
        "group": "default",
        "attributes": {
          "name": "csrf_token",
          "type": "hidden",
          "value": "WwXwxKP3M6IwsMFbexhwOS2yVt3MsdQICVhAU4MJh/7TGDxXbkDoExqP+kdKAhuUDMoc2mTcK3eRn9P/kIAlPQ==",
          "required": true,
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {}
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "traits.email",
          "type": "email",
          "value": "1@local",
          "required": true,
          "autocomplete": "email",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "E-Mail",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "traits.nickname",
          "type": "text",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "nickname",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "traits.birthdate",
          "type": "text",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "birthdate",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "method",
          "type": "submit",
          "value": "profile",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070003,
            "text": "Save",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "password",
          "type": "password",
          "required": true,
          "autocomplete": "new-password",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070001,
            "text": "Password",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "method",
          "type": "submit",
          "value": "password",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070003,
            "text": "Save",
            "type": "info"
          }
        }
      }
    ]
  },
  "identity": {
    "id": "793126a9-3c8b-43ec-89d0-e48395235131",
    "schema_id": "user_v1",
    "schema_url": "http://localhost:4533/schemas/dXNlcl92MQ",
    "state": "active",
    "state_changed_at": "2024-01-31T03:53:49.789625Z",
    "traits": {
      "email": "1@local"
    },
    "verifiable_addresses": [
      {
        "id": "a7d3f207-0a8d-47af-b0fb-576806a1bcde",
        "value": "1@local",
        "verified": true,
        "via": "email",
        "status": "completed",
        "verified_at": "2024-01-31T04:08:28.273878Z",
        "created_at": "2024-01-31T03:53:49.7915Z",
        "updated_at": "2024-01-31T03:53:49.7915Z"
      }
    ],
    "recovery_addresses": [
      {
        "id": "694551fc-4074-4b92-b8e2-8cfe0a67c2e6",
        "value": "1@local",
        "via": "email",
        "created_at": "2024-01-31T03:53:49.792294Z",
        "updated_at": "2024-01-31T03:53:49.792294Z"
      }
    ],
    "metadata_public": null,
    "created_at": "2024-01-31T03:53:49.790597Z",
    "updated_at": "2024-01-31T03:53:49.790597Z"
  },
  "state": "show_form"
}
```


#### 2. Settings flow送信API (method: profile)

endpoint: `POST {{ kratos public endpoint }}/self-service/settings`

[APIドキュメント](https://www.ory.sh/docs/kratos/reference/api#tag/frontend/operation/updateSettingsFlow)

1.で初期化したSettings flowを実行します。

ここでは、プロファイルが更新されます。

emailを更新した場合（変更があった場合）のみ、レスポンスに`continue_with`が含まれます。

`continue_with`が含まれていた場合は、[Registration flowからVerification flowへの切り替え](https://github.com/YoshinoriSatoh/kratos_selfservice_example?tab=readme-ov-file#2-registration-flow%E3%81%AE%E5%AE%9F%E8%A1%8Capimethod-password)と同様の手順で、Verification flowを実施する必要があります。

**レスポンス例**
```json
{
  "id": "38c9c3ee-45c8-4b98-9d8b-d4850f5e8408",
  "type": "browser",
  "expires_at": "2024-01-31T10:11:13.388073Z",
  "issued_at": "2024-01-31T09:11:13.388073Z",
  "request_url": "http://localhost:4533/self-service/settings/browser",
  "ui": {
    "action": "http://localhost:4533/self-service/settings?flow=fb01878b-2597-4116-ad7d-2d776dcb3cae",
    "method": "POST",
    "nodes": [
      {
        "type": "input",
        "group": "default",
        "attributes": {
          "name": "csrf_token",
          "type": "hidden",
          "value": "5n6cpiLdE4bGH6YDQDq/sZ6VTNFS8U+kAPTk4w52CkxuY1A172rIN+wgnR9xINQcv+0G1vqcsNuYM3dPHf+ojw==",
          "required": true,
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {}
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "traits.email",
          "type": "email",
          "value": "updated-1@local",
          "required": true,
          "autocomplete": "email",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "E-Mail",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "traits.nickname",
          "type": "text",
          "value": "updated-nickname",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "nickname",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "traits.birthdate",
          "type": "text",
          "value": "2000-01-01",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070002,
            "text": "birthdate",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "profile",
        "attributes": {
          "name": "method",
          "type": "submit",
          "value": "profile",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070003,
            "text": "Save",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "password",
          "type": "password",
          "required": true,
          "autocomplete": "new-password",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070001,
            "text": "Password",
            "type": "info"
          }
        }
      },
      {
        "type": "input",
        "group": "password",
        "attributes": {
          "name": "method",
          "type": "submit",
          "value": "password",
          "disabled": false,
          "node_type": "input"
        },
        "messages": [],
        "meta": {
          "label": {
            "id": 1070003,
            "text": "Save",
            "type": "info"
          }
        }
      }
    ],
    "messages": [
      {
        "id": 1050001,
        "text": "Your changes have been saved!",
        "type": "success"
      }
    ]
  },
  "identity": {
    "id": "793126a9-3c8b-43ec-89d0-e48395235131",
    "schema_id": "user_v1",
    "schema_url": "http://localhost:4533/schemas/dXNlcl92MQ",
    "state": "active",
    "traits": {
      "email": "updated-1@local",
      "nickname": "updated-nickname",
      "birthdate": "2000-01-01"
    },
    "verifiable_addresses": [
      {
        "id": "e8fc5c26-f422-4d85-8dd7-20753969b142",
        "value": "updated-1@local",
        "verified": false,
        "via": "email",
        "status": "sent",
        "created_at": "2024-01-31T09:11:13.502702Z",
        "updated_at": "2024-01-31T09:11:13.502702Z"
      }
    ],
    "recovery_addresses": [
      {
        "id": "f95f28ca-e1e3-4887-b1a6-02c05c6a8b2c",
        "value": "updated-1@local",
        "via": "email",
        "created_at": "2024-01-31T09:11:13.501268Z",
        "updated_at": "2024-01-31T09:11:13.501268Z"
      }
    ],
    "metadata_public": null,
    "created_at": "2024-01-31T03:53:49.790597Z",
    "updated_at": "2024-01-31T03:53:49.790597Z"
  },
  "state": "success",
  "continue_with": [
    {
      "action": "show_verification_ui",
      "flow": {
        "id": "bf045695-1d9d-4f19-ab8c-f4ddf0dd388d",
        "verifiable_address": "updated-1@local",
        "url": "http://localhost:8000/auth/verification?flow=bf045695-1d9d-4f19-ab8c-f4ddf0dd388d"
      }
    }
  ]
}
```

#### 3.4.5.(emailを更新した場合のみ) Verification flow

[Registration flow後のVerificaiton flowと同様です](https://github.com/YoshinoriSatoh/kratos_selfservice_example?tab=readme-ov-file#3-2%E3%81%A7%E5%AE%9F%E8%A1%8C%E3%81%95%E3%82%8C%E3%81%9Fverification-flow%E3%81%AB%E3%82%88%E3%82%8B%E3%83%A1%E3%83%BC%E3%83%AB%E3%82%A2%E3%83%89%E3%83%AC%E3%82%B9%E6%A4%9C%E8%A8%BC%E3%83%A1%E3%83%BC%E3%83%AB%E7%A2%BA%E8%AA%8D%E3%81%A8%E6%A4%9C%E8%A8%BC%E3%82%B3%E3%83%BC%E3%83%89%E5%85%A5%E5%8A%9B)
