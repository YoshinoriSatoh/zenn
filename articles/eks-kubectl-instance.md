---
title: "EKSのコンテキスト間違いを防ぐ"
emoji: "😸"
type: "tech"
topics: ["aws", "eks", "kubernetes"]
published: true
---

## 背景 - コンテキスト切り替え忘れのヒヤリハット
EKSクラスタへkubectlを実行する際、コンテキストをうっかり間違えてしまって、ひやっとした経験はありませんでしょうか？

GitOpsを構築していれば、kubectlで直接apply等の操作を行う機会はなくなるはずではありますが、とはいえいざという時のために、kubectlによる操作環境も構築しておきたいところです。

なるべく安全にkubectlを操作できる環境を構築してみました。

## 複数クラスターを扱うkubectl実行環境

EKSのAPIサーバは、パブリック・プライベートもしくはその両方のアクセス許可を設定可能です。

プライベートアクセスを許可している場合はVPC内のEC2インスタンスやVPN接続から、パプリックアクセスも許可している場合はローカルマシン等からインターネット経由で、kubectlを実行できます。

アクセスポリシーによってkubectlを実行する環境は変わりますが、いずれも一つのマシンにて複数のEKSクラスターを切り替えて実行可能です。

このため、ユーザごとの権限さえ適切に管理されていれば、kubectl実行時にコンテキストを切り替えることで、実行環境の要件としては十分そうにみえます。

![](https://storage.googleapis.com/zenn-user-upload/cgq8dj02h6gqlwub6dq4jubevbpc)

## 問題点

ここで以下の課題が見えてきました。

* 手動コンテキスト切り替えミスの可能性
* kubernetesバージョンに応じたkubectlへの切り替えミスの可能性

![](https://storage.googleapis.com/zenn-user-upload/q6sbd0jigxyd2xqw5fvbx0piwey1)

### 手動コンテキスト切り替えミスの可能性

上記の環境では、コンテキストの切り替えは手動で行う必要があります。

このコンテキスト切り替えは絶対に間違ってはならず、コンテキストを間違ってapplyしてしまうと、サービスがダウンしたり、突然別バージョンで起動するなど、広範囲にわたって被害が及びます。

今現在どのコンテキストであるかは、勿論kubectlで確認可能ですが、これも手動操作・手動確認です。

kubectlで更新系作業を実施する際には、複数人で手順やdry-runのレビューを必須にする等の対策も考えられます。

このような対策が上手くワークする状況であれば、その方が適しているかもしれませんが、本記事ではそうでない状況において検討しています。

### kubernetesバージョンに応じたkubectlへの切り替えミスの可能性

kubernetesバージョンが環境毎に異なる状況も考えられます。

例えば、develop環境で一つ新しいバージョンへ更新して、ある程度の検証期間を経てproductionへ適用するような状況です。

この場合、コンテキストの他に、kubectlのバージョンも合わせて切り替える必要があります。

## 解決案 - クラスター個別のkubectl専用インスタンス
検討した結果、泥臭い手法ではありますが、以下のようにEKSクラスター毎にkubectl専用のEC2インスタンス(以下kubectlインスタンス)を構築するようにしてみました。

![](https://storage.googleapis.com/zenn-user-upload/knukrvuos0w4vlk833pb90vxmb9c)

要点は以下の通りです。

* kubectlインスタンスは対応する一つのクラスターへのコンテキストしか持たない
* クラスターのadmin権限はkubectlインスタンスにしか持たせない (systems:master)
* 各IAMユーザは閲覧のみ可能な権限を割り当てる (viewer)
* kubectlインスタンスはprivate subnetに配置し、インターネットインバウンドを持たない
* kubectlインスタンスにはSSM Session Managerで接続する
* kubectlインスタンスは、EventBridgeによって日次でスケジュール停止する

## そもそもコンテキスト切り替えの必要性をなくす

kubectlインスタンスは、対応する一つのクラスターへのコンテキストしか持ちません。

このため、そもそもコンテキストやkubectlの切り替えミスのリスクがありません。

ログインするkubectlインスタンスを間違えてしまう、コンテキストは正しいが適用マニフェストを間違えてしまう、といった事故も依然として起こりえますが、このリスク低減については後述します。

## EKSクラスターのRBAC

EKSクラスターでは、KubernetesのClusterRole/ClusterRoleBinding と AWS IAMロール/IAMユーザとのマッピングが可能です。

マッピングはaws-authコンフィグマップに記述します。

デフォルトで、クラスターの管理者権限グループとして`systems:master`が存在しており、kubectlインスタンスロールを紐づけています。

また、図中の`viewer`グループは、別途作成した閲覧権限のみが付与されたグループで、各IAMユーザと紐づけています。

IAMユーザにKubernetesリソースの更新系の権限を付与した運用も可能ですが、これだと今回の課題であるコンテキスト切り替えミスのリスクを抱えることになります。

運用の仕方にもよりますが、強い権限をkubectlインスタンスロールに限定することで、セキュリティリスクも低減できそうです。

## kubectlインスタンスへのログイン(SSM SessionManager)

kubectlインスタンスには、SSM SessionManagerでログインさせます。

SessionManagerを利用することで、以下利点があります。

* インスタンスにインターネットインバウンドが不要
* IAMユーザー/グループ単位でインスタンスへのアクセス権限管理が可能
* IAMユーザー単位でアクティビティログ（操作ログ）をセキュアに保存可能

### インスタンスにインターネットインバウンドが不要

前述の通り、kubectlインスタンスロールはEKSクラスターに対して強い権限を持っているため、kubectlインスタンスのセキュリティには配慮が必要です。

SessionManagerでのログインに限定することで、インターネットインバウンドが必要がなくなるため、インターネット経由の攻撃リスクがなくなります。

あとは、IAMユーザーのセキュリティを確保(MFAを必須にするなど)することで、かなりセキュアな環境を構築できます。

### IAMユーザー/グループ単位のアクセス管理

kubectlインスタンスへログインできるIAMユーザーを限定する必要があります。

IAMユーザーおよびIAMグループに付与されているIAMポリシーによって、kubectlインスタンスへのアクセス権限を設定可能です。

(SessionManagerのRunAsを有効化している場合は、OSユーザーレベルで制御するパターンもあるかもしれません)

### IAMユーザー単位のアクティビティログ（操作ログ）

SessionManagerでのアクティビティは、Coudwatch LogsおよびS3へ保存可能です。

誰がどのような操作を行なったのかを監査可能です。

これらのログは、kmsを使用した暗号化も可能です。

### EventBridgeによるスケジュール停止

上記のようにkubectl環境を構築しましたが、kubectlインスタンスは普段はそれほど使用しないため、日次で停止するようにEventBridgeを組んでいます。

EventBridgeから、EC2インスタンスのstop-instances APIを呼び出すことができます。

EventBridgeの画面だけで設定可能であるため非常にお手軽ですが、その反面EC2インスタンスIDの指定しかできません。

kubectlインスタンスの数が増えてくると少々管理が面倒になるかもしれません。

その場合は、Automationを活用して、指定したタグのインスタンスを対象にすることもできるようです。

以下参考記事です。

https://qiita.com/uzresk/items/504d52bf1042c1067dde

## 残るリスクへの対策案

上記のように、コンテキスト切り替えの間違いによるリスクを排除したkubectl実行環境ですが、依然として以下のリスクが存在します。

* ログイン先kubectlインスタンスを間違える(気がつかずに作業)
* 対象のクラスターに別環境のマニフェストを適用してしまう

このリスク低減についても対策してみました。

### ログイン先kubectlインスタンスを間違える(気がつかずに作業)

kubectlインスタンスのログイン時に、視覚的に目立つメッセージを表示して、間違いに気付かせるアプローチです。

Amazon Linux2へのSSHログインの際には、デフォルトで以下のようなメッセージが表示されます。

```

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
```

メッセージは`/etc/motd`ファイルに記述されているのですが、SessionManagerは厳密にはSSHログインではないため、これが読み込まれません。

`.bashrc`で読み込ませれば良さそうなものですが、上記と同様の理由で`.bashrc`も読み込まれません。

このため、SessionManagerの"Shell Profile"を活用して、ログインシェルを読み込んでいます。

```
sudo -iu `whoami`
```

![](https://storage.googleapis.com/zenn-user-upload/xgcytypaaskpgkvwoulwrgbkzzf0)

この上で`.bashrc`に以下を追記します。

```
cat /etc/motd
```

ssm-userとシェルの読み込みについては以下記事が詳しいのでご参照ください。

https://dev.classmethod.jp/articles/mitigation-idea-for-session-manager/

`/etc/motd`をカスタムして、どのクラスターに対するkubectlインスタンスにログインしたのかをわかりやすくしています。

```

This is  DEVELOP  kubectl instance.

```

例えば本番環境のクラスターには、以下のように警告的なメッセージにするのもいいと思います。

```

!!! WARNING !!!
This is  PRODUCTION  kubectl instance.
!!! WARNING !!!

```

もちろんこれで100%防げるわけではなく、ログイン先のインスタンスを間違えてしまう可能性は残りますが、かなり間違いに気が付きやすくなると思います。

### 対象のクラスターに別環境のマニフェストを適用してしまう

手動操作では、「コンテキストは正しいが適用するマニフェストを間違ってしまう」ということも考えられます。

kustomizeベースのマニフェストの場合、例えば以下のようなディレクトリ構成になります。

```
.
├── base
└── overlays
    ├── develop
    └── prod
```

この場合、overlays配下の`develop`と`prod`はそれぞれ対応するクラスターに必要なものですが、kubectl実行時に移動するディレクトリを間違えてしまい、prod環境に対してdevelopマニフェストを適用してしまう事故も起こりえます。

せっかくkubectlインスタンスをクラスター個別に用意しているので、対象外のクラスター用のマニフェストをkubectlインスタンスから削除することで、この事故をなくせるはずです。

GitOpsを構築している前提なので、マニフェストはgitリポジトリからcloneしてくることになりますが、clone後にoverlays配下の不要なディレクトリを削除します。

## まとめ

複数クラスターに対する更新系操作のkubectl環境について、手動操作時のコンテキスト切り替えに関するリスクと解決案を記述しました。

手動操作を完全に無くせるのが最も良い方法とは思いますが、私自身まだまだkubernetes力が足りず、kubectl環境がないと正直不安です。

安全なkubectl環境を用意しようと思うと、結構考えることが多く、結果泥臭い手法となってしまった感があります。

もっとスマートな解決策があれば是非ともご教授くださいm(_ _)m
