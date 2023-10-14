---
title: ルートテーブルとサブネットの明示的な関連付けの罠
tags:
  - AWS
  - CLI
  - vpc
private: false
updated_at: '2023-10-14T23:09:23+09:00'
id: ddaedcd6946bd4375a32
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに
<!-- 発端や概要を記載 -->
皆様はルートテーブルとサブネットの明示的な関連付けの仕様について、正確に理解していますか？？
どちらも AWS が提供しているネットワーク系リソースの中で基礎となるものであり、なおかつ手軽に作成できるので、ドキュメントを隅々まで読み仕様を把握している人は案外少ないのかもしれません。

そんなルートテーブルとサブネットについて、凄まじい罠を発見しました。
知ってる人からしたらかなり常識的なことですが、案外見落としがちな仕様だと感じたので共有させてください...。

# 目次
<!-- タイトルとアンカー名を編集 -->
1. [サブネットとルートテーブルの関係性について](#1-サブネットとルートテーブルの関係性について)
2. [関連付けをするまでの作業](#2-関連付けをするまでの作業)
3. [一体何が罠なのか](#3-一体何が罠なのか)
4. [罠を回避するには](#4-罠を回避するには)
5. [最後に](#5-最後に)

<!-- 各チャプター -->
<a id="#Chapter1"></a>
# 1-サブネットとルートテーブルの関係性について

まずサブネットとルートテーブルの関連付けについて、以下のような仕様があります。

```
(1)サブネットに明示的にルートテーブルの関連付けを設定しない場合、自動的にデフォルトのルートテーブルが適用される。
(2)サブネットに明示的に関連付けられるルートテーブルは、1つのみ。
(3)ルートテーブルは複数のサブネットと明示的な関連付けができる。
```

つまりサブネットから見たルートテーブルは`1対1`の関係、
ルートテーブルから見たサブネットは`1対多`の関係となります。

[ルートテーブル-公式ドキュメント](https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/WorkWithRouteTables.html#DisassociateSubnetRouteTable)

<a id="#Chapter2"></a>
# 2-関連付けをするまでの作業

次に、サブネットとルートテーブルの明示的な関連付けを行う際は以下の作業が発生します。

```
(1)サブネットを作成
(2)ルートテーブルを作成
(3)以下どちらかの方法で、サブネットとルートテーブルを明示的に関連付ける
   -1.サブネット側から関連付けを行う
   -2.ルートテーブル側から関連付けを行う
```

ここまで見た方の中ですでにお察しの方がいるかもしれませんが(3)-2の作業について、
「1 つの VPC 内に複数のサブネットが存在し、なおかつ共通のルートテーブルを利用している」環境の場合、とんでもない罠が潜んでいます。

<a id="#Chapter3"></a>
# 3-一体何が罠なのか

`ルートテーブル側から、関連付けるサブネットを設定する`場合の作業について、
具体的には以下の手順で実行します。

#### (1) ルートテーブルから「サブネットの関連付けを編集」を実行
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/d0d6d043-c23d-0971-375b-e7c85aa7585f.png)
#### (2) 関連付けを行うサブネットを選択して、「関連付けを保存」を実行
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/ddf620a0-b0d1-742b-3ed3-69f30276836a.png)
#### (3) 「明示的なサブネットの関連付け」が更新される
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/77c96a5a-cf8f-71a0-0b61-1d52c3c7bc1b.png)

お分かりいただけましたでしょうか。
ルートテーブル側からは複数のサブネットを選択出来るため、誤って関連付け対象外のサブネットを選んでしまった場合、
元々明示的に関連付けされているルートテーブルが変更されます。
つまり`サブネットに明示的に関連付けられるルートテーブルは一つのみ`なので、
`既存のルートテーブルが外れて、誤ったルートテーブルが関連付けられてしまいます。`

もし上記画像の通り、`誤って選択したのがプライベートサブネット`で、
`誤って関連付けをしたルートテーブルのルートに、インターネットゲートウェイを含んでいたら？？？`

想像するだけで震えます...。


<a id="#Chapter4"></a>
# 4-罠を回避するには

このような悲劇を防ぐには、主に以下二通りのアプローチが考えられます。
今となってはベストプラクティス的な内容ですが、
本記事の設定以外にも通ずる内容だと思うので、今一度確認してみてください。

## 構成上の回避策
#### (1)アカウントの分離
可能であればシステム単位でAWSアカウントを分けましょう、
そしてAWSアカウントの管理はAWS Organizationsにて管理をしましょう。
[AWSアカウントの管理と分離-公式ドキュメント](https://docs.aws.amazon.com/ja_jp/wellarchitected/latest/security-pillar/aws-account-management-and-separation.html)

ルートテーブルやサブネットに限った話ではないですが、作成したリソースは当然アカウントに所属するため、
アカウントを分離してしまえば、個別にIAM設定などしない限りは別アカウントのリソースに対するアクションは行えません。

#### (2)ネットワーク関連リソースの分離
VPC以降のリソース(サブネット、ルートテーブル、NACLなど)をシステム単位で分離しましょう。
また可能であればVPC単位で分離するべきですが、`クォータの問題`や、
`システム間の通信が必要な場合は、 VPCピアリング、Transit Gateway、PrivateLinkなどの設定が必要となる`ので、
要件や費用に応じて要検討になるかと思います。
[VPCクォータ-公式ドキュメント](https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/amazon-vpc-limits.html)
[VPCピアリング-公式ドキュメント](https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/vpc-peering.html)
[Transit Gateway-公式ドキュメント](https://docs.aws.amazon.com/ja_jp/vpc/latest/tgw/what-is-transit-gateway.html)
[PrivateLink-公式ドキュメント](https://docs.aws.amazon.com/ja_jp/vpc/latest/privatelink/privatelink-share-your-services.html)


## 運用上の回避策
#### (1)サブネットから関連付けを行う
[一体何が罠なのか](#3-一体何が罠なのか)の手順ではルートテーブル側から関連付けの設定を実行しましたが、
サブネット側からの関連付けも可能です。
サブネットから選ぶ場合は、選択できるルートテーブルは一つのみのため、
誤って他のサブネットに影響を及ぼすことはありません。
#### (1) サブネットから「ルートテーブルの関連付けを編集」を実行
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/a08c759c-3d8a-8204-abd4-20b188d0bd8c.png)
#### (2) 関連付けを行うルートテーブルを選択して、「保存」を実行
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/bbccb1f2-bc5a-e1ce-fba2-cc62e845f463.png)
#### (3) 「ルートテーブル」が更新される
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/7fb67d05-125b-b517-598c-fd6a90037c51.png)

ただし、そもそもGUIで作業すること自体がよろしくないので、後述(2),(3)がおすすめです。

#### (2)AWS CLIの利用
以下コマンドで、サブネットとルートテーブルを指定して関連付けが実行できます。
ただしIDの確認は別のコマンドで確認が必要となり手順やその後の運用が煩雑になるので、後述(3)が一番おすすめです。
```bash
aws ec2 associate-route-table \
  --subnet-id $Subnet_ID \
  --route-table-id $RouteTable_ID
```
[AWS CLI-公式ドキュメント](https://docs.aws.amazon.com/cli/latest/reference/ec2/associate-route-table.html)

#### (3)IaCツールの利用
おとなしくCloudFormationやTerraformなどのIaCツールを導入しましょう。
導入にあたり学習コストや、運用方式の定義などの人的コストはかかりますが、
長い目で見た場合、最終的にはメリットのほうが大きいと思います。

最近ライセンス変更が発生して若干雲行きが怪しいところもありますが、
マルチクラウドに対応している点から個人的にはTerraformを推奨します。

[AWS CloudFormation-公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/Welcome.html)
[Terraform-公式ドキュメント](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

<a id="#Chapter5"></a>
# 5-最後に
若干長めの記事になってしまいましたが、最後までお付き合いいただきありがとうございます。
願わくばこの記事が、システム構築時や運用後にて発生する事故の削減に貢献できることを祈っています。
