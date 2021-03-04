---
title: "ECS FargateプラットフォームバージョンのLATESTが1.4.0へ"
emoji: "😸" 
type: "tech" 
topics: ["aws", "ecs", "fargate"] 
published: false 
---

## ECS Fargate PV LATESTの変更アナウンス
AWSより以下アナウンスがありました。

```
2021 年 3 月 8 日より、AWS Fargate はプラットフォームバージョン (PV) LATEST フラグを変更して、PV 1.3.0 [1] ではなく PV 1.4.0 に解決します。
PV 1.4.0 は、2020 年 4 月に開始され、多くの新機能や変更点を提供しています [2]。パブリックインターネットアクセスを持たない VPC でタスクを実行している場合は、アクションが必要です。
```

ECS Fargateでは、プラットフォームバージョンというものが存在します。

上記の通り、2021年3月8日から、このプラットフォームバージョン(PV)のLATESTが1.4.0となります。

PV1.4.0自体は、結構前からサポートされていましたが、これまでLATESTが1.3.0のままだったので、なんだか違和感がありました。

## PV1.3.0と1.4.0の互換性

PV1.3.0と1.4.0では、以下の点で**互換性がありません。**

* ECR VPCエンドポイントが必要 (ECRを使用する場合)
* Secrets Manager VPCエンドポイントが必要 (Secrets Managerを使用する場合)
* Systems Manager VPCエンドポイントが必要 (Systems Managerを使用する場合)

上記要件を満たせていないと、1.4.0にプラットフォームバージョンが上がった際に、タスクが起動できなくなります。

私は普段ECSを構築する際に、なかなかVPCエンドポイントまでは構築していなかったので、試しに1.4.0に上げてみたところ、タスクの起動に失敗してしまいました。

## PV1.4.0に更新するべきか
VPCエンドポイントは作成しているだけで、料金が発生します。

あまり使っていない場合でも、一つのVPCエンドポイントにつき大体10USD程度かかります。

さらに、一つだけでなく複数のVPCエンドポイントが必要かつ、VPC毎にエンドポイント群を作成する必要があると考えると、VCPエンドポイントのコストが結構かさんできます。

NLBを使用したいなど、PV1.4.0でなければ満たせない要件がある場合には、これらのコストも予め見積もっておく必要があります。

1.3.0で十分要件が満たせるのであれば、現状では1.4.0にこだわる必要はなさそうに思います。

## 参考
AWS Fargate プラットフォームのバージョン
https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/userguide/platform_versions.html

AWS Fargate for Amazon ECS が Network Load Balancer を使用した UDP ロードバランシングのサポートを開始
https://aws.amazon.com/jp/about-aws/whats-new/2020/07/aws-fargate-for-amazon-ecs-now-supports-udp-load-balancing-with-network-load-balancer/