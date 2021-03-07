---
title: "ECS FargateでVPCエンドポイントを使用する"
emoji: "😸" 
type: "tech" 
topics: ["aws", "ecs", "fargate", "vpc", "terraform"] 
published: true 
---
## 概要＆背景
以下記事にて、ECS FargateでVPCエンドポイントが必要となる状況について、誤った理解をしていたこともあり、改めてECS FargateとVPCエンドポイントの関係や動作検証をしてみました。

https://zenn.dev/yoshinori_satoh/articles/ecs-fargate-latest-update

また、既に十分解説されている記事も存在していますので、そちらもご参照ください。

https://dev.classmethod.jp/articles/privatesubnet_ecs/

本記事の内容は、上記記事と重複も多いですが、検証環境を再現可能なTerraformコードも合わせて公開しています。

## ECS FargateとVPCエンドポイントの関係

Fargateに限らず、ECSではアプリケーションで行う通信以外に、以下の通信が発生します。

（それぞれ必ず発生するわけではなく、ECR等の対応AWSリソースを使用する場合のみ通信が発生します）

* ECRからDockerイメージPULL 
* Cloudwatch Logsへログ出力
* タスク定義からParameterStore参照
* タスク定義からSecretsManager参照

インターネットへのアウトバウンドを持つサブネット、かつVPCエンドポイントがない場合は、アプリケーションの通信も上記の通信も、どちらもインターネットを経由した通信となります。

以下は、NATを構成した場合の上記の通信経路イメージです。

![](https://storage.googleapis.com/zenn-user-upload/r3f2h6ybtrq3aqho3vax66ncfllg)

上記のように、AWSリソースヘの通信をインターネットを経由させたくない場合や、そもそもインターネットから完全に切り離した独立したサブネットでタスクを起動したい場合には、VPCエンドポイントを構成する必要があります。

VPCエンドポイントを構成した場合の通信経路イメージは以下のようになります。
（インターネットから切り離したサブネットの場合）

![](https://storage.googleapis.com/zenn-user-upload/iuviwc4m3s934nqy76hsbf1glr9m)

## VPCエンドポイントとPrivateLink

VPCエンドポイントとAWSサービスとの通信は、Amazonのネットワーク内で完結します。 

これは PrivateLink と呼ばれます。

上図では、S3以外のサービスはPrivateLinkで通信されています。

VPCエンドポイントにはいくつか種類があり、PrivateLinkを構成可能なものは「インターフェイスエンドポイント」です。

インターフェイスエンドポイントの実態は、ENI(Elastic Network Interface)であり、サブネットと紐付けられます。

S3のみ「ゲートウェイエンドポイント」です。

ECS Fargateを使用する際のECRのVPCエンドポイントに関するドキュメント上、S3についてはゲートウェイエンドポイントを使用するよう記述されています。

S3のインターフェイスエンドポイントは、つい最近までサポートされていなかったようですが、以下記事によると2021年2月3日にサポートされたようです。
(Terraformはまだ未対応のようです)

https://dev.classmethod.jp/articles/private-link-for-s3/

しかし、ゲートウェイエンドポイントは料金もかからないため、この用途で使用する場合には、ゲートウェイエンドポイントのままで良さそうです。

ゲートウェイエンドポイントもインターフェイスエンドポイントと同様に、Amazonネットワークに閉じた通信となります。

## ECS Fargateで必要なVPCエンドポイント

上記の図中にも記載していますが、各ユースケースに対応する以下のVPCエンドポイントが必要です。

(PV=PlatformVersion)

| ユースケース | PV 1.3.0 | PV 1.4.0 |
| --- | --- | --- |
| ECRからDockerイメージPULL | s3(Gateway) <br /> ecr.dkr | s3(Gateway) <br /> ecr.dkr <br /> ecr.api |
| Cloudwatch Logsへログ出力 | logs | logs | 
| タスク定義からParameterStore参照 | ssm | ssm |
| タスク定義からSecretsManager参照 | secretsmanager | secretsmanager |

## Terraformコード

上図の「VPCエンドポイントを構成した場合の通信経路イメージ」を構築するTerraformコードを公開しています。

https://github.com/YoshinoriSatoh/terraform-sandbox/tree/ecs-fargate-vpc-endpoint-1/environments/ecs-fargate-vpc-endpoint

インターネットゲートウェイにも、NATも構成されていないサブネットを用意しています。

```
resource "aws_subnet" "private-a" {
  vpc_id            = aws_vpc.main.id
  availability_zone = "${data.aws_region.current.name}a"
  cidr_block        = var.subnets.private.a.cidr_block
  tags = merge(
    local.common_tags,
    {
      Name = "${local.infra_fullname}-private-a"
    }
  )
}

resource "aws_route_table" "private-a" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    local.common_tags,
    {
      Name = "${local.infra_fullname}-private-a"
    }
  )
}

resource "aws_route_table_association" "private-a" {
  subnet_id      = aws_subnet.private-a.id
  route_table_id = aws_route_table.private-a.id
}
```

このサブネットおよびルートテーブルに対して、以下のVPCエンドポイント群を作成しています。
```
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${data.aws_region.current.name}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids = [ aws_route_table.private-a.id ]
}

resource "aws_vpc_endpoint" "ecr-dkr" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${data.aws_region.current.name}.ecr.dkr"
  vpc_endpoint_type = "Interface"
  private_dns_enabled = true
  subnet_ids = [ aws_subnet.private-a.id ]
  security_group_ids = [ aws_security_group.vpc-endpoint.id ]
}

resource "aws_vpc_endpoint" "ecr-api" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${data.aws_region.current.name}.ecr.api"
  vpc_endpoint_type = "Interface"
  private_dns_enabled = true
  subnet_ids = [ aws_subnet.private-a.id ]
  security_group_ids = [ aws_security_group.vpc-endpoint.id ]
}

resource "aws_vpc_endpoint" "secretsmanager" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${data.aws_region.current.name}.secretsmanager"
  vpc_endpoint_type = "Interface"
  private_dns_enabled = true
  subnet_ids = [ aws_subnet.private-a.id ]
  security_group_ids = [ aws_security_group.vpc-endpoint.id ]
}

resource "aws_vpc_endpoint" "ssm" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${data.aws_region.current.name}.ssm"
  vpc_endpoint_type = "Interface"
  private_dns_enabled = true
  subnet_ids = [ aws_subnet.private-a.id ]
  security_group_ids = [ aws_security_group.vpc-endpoint.id ]
}

resource "aws_vpc_endpoint" "logs" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${data.aws_region.current.name}.logs"
  vpc_endpoint_type = "Interface"
  private_dns_enabled = true
  subnet_ids = [ aws_subnet.private-a.id ]
  security_group_ids = [ aws_security_group.vpc-endpoint.id ]
}
```

それぞれ`private_dns_enabled = true`で、プライベートDNSを有効化しています。

ecr.dkr と ecr.api については、以下ドキュメントに有効化する旨の記載があります。

https://docs.aws.amazon.com/ja_jp/AmazonECR/latest/userguide/vpc-endpoints.html

それ以外のエンドポイントについては記述を見つけられなかったのですが、どうやら全てのエンドポイントのプライベートDNSを有効化する必要があるようです。

試しに、secretsmanagerのVPCエンドポイントのプライベートDNSを無効化したところ、タスク起動時に以下エラーとなりました。

```
ResourceInitializationError: unable to pull secrets or registry auth: execution resource retrieval failed: unable to retrieve secret from asm: service call has been retried 1 time(s): failed to fetch secret arn:aws:secretsmanager:ap-northeast-1:5394593...
```

## まとめ
VPCエンドポイントとECS Fargateとの関係について説明しました。

インターフェイスエンドポイントとゲートウェイエンドポイントの基本的事項およびPrivateLinkについても簡単に説明しました。

Terraformによる再現可能な検証コードも公開していますので、参考になりますと幸いです。

何か間違っていたり、補足などあればコメントいただけると大変ありがたいですm(_ _)m

## 参考
https://dev.classmethod.jp/articles/privatesubnet_ecs/

https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/vpc-endpoints.html

https://docs.aws.amazon.com/ja_jp/AmazonECR/latest/userguide/vpc-endpoints.html