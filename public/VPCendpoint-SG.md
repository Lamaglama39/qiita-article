---
title: セキュリティーグループのアウトバウンドルールでVPCエンドポイント宛の通信を許可する方法
tags:
  - 'AWS'
  - 'vpcendpoint'
  - 'SecurityGroup'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに
<!-- 発端や概要を記載 -->
セキュリティーグループのアウトバウンドは設定しないことが多いですが、
セキュリティー要件が厳しいシステムだと細かく制御を求められることがあります。

そんなとき、
「VPCエンドポイント宛の通信ってどうやって指定するんだ？？」、
「そもそもVPCエンドポイントにIPアドレスあったっけ？？」なんてお悩みがわきませんか？？

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/eb97e851-eaf5-aacf-76ab-c76db630201e.png)

本記事はそんなお悩みをもった方に向けたものです。

また以降の図ではクライアント側はEC2としていますが、ECSやLambda(VPC内)などセキュリティーグループをアタッチできるリソースは同様の設定方法となります。

<!-- 各チャプター -->
<a id="#Chapter1"></a>

# 1.インターフェース型VPCエンドポイントの場合

## ・VPCエンドポイントのIPアドレスを指定する
まずは素直にIPアドレスを指定する方法です。
インターフェース型のVPCエンドポイントは作成時に指定したサブネット内にてENIが作成されVPCエンドポイントに割り当てられるので、ENIのプライベートIPアドレスを指定することができます。
またこのENIはVPCエンドポイントを削除しない限りは消えないため、IPアドレスが変化することもありません。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/a0169101-c645-08d9-d6d4-7b3fcac12e66.png)

またVPCエンドポイントを複数作成する場合は、`VPCエンドポイント用のサブネットを用意してサブネットのIP CIDRごと許可する`、といったアプローチも考えられます。
こうすることで設定値の煩雑さを多少軽減できます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/3edaaa8b-57f9-7f71-8626-4809d8824a4a.png)

## ・VPCエンドポイントのセキュリティーグループを指定する
次に`VPCエンドポイントに割り当てたセキュリティーグループのID`を指定する方法です。
個別にIPアドレスを設定する必要もなく、
他のVPCエンドポイントとセキュリティーグループを共有することもできるので、
IPアドレス指定に対する熱いこだわりがなければ、この方法がおすすめです。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/981c7e72-7473-a534-2b4d-29275f01eff4.png)


<a id="#Chapter2"></a>

# 2.ゲートウェイ型VPCエンドポイントの場合

## ・マネージドプレフィックスリストを指定する
ゲートウェイ型VPCエンドポイントはENIは存在せず、宛先がグローバルIPアドレスとなります。
例として、S3で利用されているIP CIDRのリストは以下ドキュメントに記載があります。
* [AWS IP アドレスの範囲-公式ドキュメント](https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/aws-ip-ranges.html#aws-ip-download)

愚直に上記でIP CIDRを調べて設定してもいいですが、
AWS側が予め各IP CIDRを設定したマネージドプレフィックスリストを用意してくれているので、ありがたく使いましょう。
* [マネージドプレフィックス-公式ドキュメント](https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/working-with-aws-managed-prefix-lists.html)

セキュリティーグループの送信先としてS3のマネージドプレフィックスリストのIDを指定すれば、一括でS3で利用されているIP CIDRを許可できます。
余談ですがマネージドプレフィックスリストはリージョンごとに固定であり、今現在東京リージョンのIDは`pl-61a54008`です。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/c04c372e-31ff-9807-9aee-fc65e02a360d.png)
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/1d482a1f-3eaa-f413-7954-0bb131f81d44.png)


<a id="#Chapter3"></a>

## 3-最後に
以上、セキュリティーグループのアウトバウンドルールでVPCエンドポイントを指定する方法でした。

また本記事では記載しませんでしたが、
`VPCエンドポイント側のセキュリティーグループのインバウンドルールで、接続元のIPアドレスを設定する`ということもできます。
正直、個人的にはそこまで細かく制御する意味は薄いと思いますが、
セキュリティー要件の厳しい案件に出会った際は設定を検討してみてください。