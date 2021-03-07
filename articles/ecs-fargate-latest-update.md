---
title: "ECS FargateプラットフォームバージョンのLATESTが1.4.0へ"
emoji: "😸" 
type: "tech" 
topics: ["aws", "ecs", "fargate"] 
published: true 
---

## 訂正
本記事を最初に投稿した時点では、私の理解が誤っており以下のような記載をしていました。

「ECS Fargate PV 1.4.0では、どんなタスクにも必ずVPCエンドポイントが必要となる」

正しい理解は以下となります。

「ECS Fargateでは、プライベートサブネットで実行され、かつインターネットへのアウトバウンドがない場合に、VPCエンドポイントが必要であり、PV1.4.0では設定が必要なVPCエンドポイントが増えた」

誤った知識を投稿してしまい申し訳ありませんでしたm(_ _)m

そして、そもそももっとわかりやすい解説記事がありましたm(_ _)m

https://dev.classmethod.jp/articles/fargate_pv14_vpc_endpoint/

改めて検証し、理解ができましたので、上記や他の記事と同じような内容になってしまいますが、得られた理解や知識を記述したいと思います。

## ECS Fargate PV LATESTの変更アナウンス
AWSより以下アナウンスがありました。

```
2021 年 3 月 8 日より、AWS Fargate はプラットフォームバージョン (PV) LATEST フラグを変更して、PV 1.3.0 [1] ではなく PV 1.4.0 に解決します。
PV 1.4.0 は、2020 年 4 月に開始され、多くの新機能や変更点を提供しています [2]。パブリックインターネットアクセスを持たない VPC でタスクを実行している場合は、アクションが必要です。
ECR [3]、Secrets Manager [4]、およびパラメータストア [5] の VPC エンドポイントをそれぞれ作成する必要があります。
```

ECS Fargateでは、プラットフォームバージョンというものが存在します。

上記の通り、2021年3月8日から、このプラットフォームバージョン(PV)のLATESTが1.4.0となります。

PV1.4.0自体は、結構前からサポートされていましたが、これまでLATESTが1.3.0のままだったので、なんだか違和感がありました。

## VPCエンドポイントの設定が必要な状況

インターネットへのアウトバウンドを持たないサブネットでタスクを起動する場合には、VPCエンドポイントが必要です。

パプリックサブネットや、NATを構成しているプライベートサブネットでは、VPCエンドポイントは必須ではありません。

（勿論、不要なトラフィックをインターネットに流さないために、VPCエンドポイントを設定しても構いません）

タスク起動時に以下の通信が発生するため、それぞれに応じたVPCエンドポイントが必要です。

* ECRからDockerイメージPULL
* Cloudwatch Logsへログ出力
* タスク定義からParameterStore参照
* タスク定義からSecretsManager参照

## PV1.3.0と1.4.0で設定が必要なVPCエンドポイントの違い

PV1.4.0では、ECRからDockerイメージをPULLするために必要なVPCエンドポイントが増えています。

| 通信 | PV 1.3.0 | PV 1.4.0 |
| --- | --- | --- |
| ECRからDockerイメージPULL | s3(Gateway) <br /> ecr.dkr | s3(Gateway) <br /> ecr.dkr <br /> **ecr.api** |
| Cloudwatch Logsへログ出力 | logs | logs | 
| タスク定義からParameterStore参照 | ssm | ssm |
| タスク定義からSecretsManager参照 | secretsmanager | secretsmanager |

`ecr.api`のVPCエンドポイントがない状態で、PV1.4.0にてタスク起動しようとすると、以下のようなエラーになります。

```
ResourceInitializationError: unable to pull secrets or registry auth: execution resource retrieval failed: unable to retrieve ecr registry auth: service call has been retried 1 time(s): RequestError: send request failed caused by: Post https://api.ecr....
```

## 参考
https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/userguide/platform_versions.html

https://aws.amazon.com/jp/about-aws/whats-new/2020/07/aws-fargate-for-amazon-ecs-now-supports-udp-load-balancing-with-network-load-balancer/

https://dev.classmethod.jp/articles/fargate_pv14_vpc_endpoint/