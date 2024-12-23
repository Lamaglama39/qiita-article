---
title: Terraformを利用したEFSのフェイルオーバー/フェイルバックについて
tags:
  - 'AWS'
  - 'EFS'
  - 'Terraform'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- 発端や概要を記載 -->
# はじめに
EFSでフェイルバックが実装されてから約1年経ちましたが、
Terraformでの切り替え手順を示している事例が見当たらなかったので記事にしました。
* [Amazon EFS でのレプリケーションフェイルバックの導入と IOPS の増加](https://aws.amazon.com/jp/blogs/news/replication-failback-and-increased-iops-are-new-for-amazon-efs/)

切り替え時間なども含めて全体の流れを確認しているので、EFSのレプリケーションを検討している方の参考になればと思います。

:::note info
本記事の内容は以下リポジトリで気軽に試せるので、よければご利用ください。
* [multi-region-efs](https://github.com/Lamaglama39/terraform-for-aws/tree/main/app/multi-region-efs)
:::

<!-- 各チャプター -->
<a id="#Chapter1"></a>

# EFS フェイルオーバー/フェイルバックの流れ
まず全体の流れについて説明します。
前提として以下のような構成のシステムを想定します。

* Region A がプライマリー、Region B がセカンダリー
* EFSのレプリケーション設定を作成済み
* 各リージョンのEC2でEFSをマウント済みで利用できる状態

![multi-region-efs.drawio.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/ed9c6bb4-796f-e03e-2b3b-a581dfe28de6.png)

### ➀フェイルオーバー
フェイルオーバーは非常に簡単で、既存のレプリケーション設定を削除するだけで完了します。
ここでは何らかの障害で Region A のEC2、またはEFSが利用不可になったと想定します。
![multi-region-efs-1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/78d8c36d-28af-e9aa-5e8f-44f36b117bd9.png)

#### (1).レプリケーション設定削除
Region A → Region B のレプリケーション設定を削除します。
![multi-region-efs-2.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/0d49c402-1f2c-9b68-cc9c-e4dc4ec098be.png)

#### (2).EFSが利用できることを確認
レプリケーション設定の削除が完了した段階で Region B のEFSが書き込み可能となるので、
EC2から利用可能なことを確認します。
![multi-region-efs-3.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/3619a9ea-cb1e-2644-8ffe-d3706c5a7e1d.png)

### ➁フェイルバック
フェイルオーバーと同様でレプリケーション設定の作成/削除の繰り返しとなります。

#### (1).レプリケーション設定作成
Region A のリソースが復旧したら、`Region B → Region A のレプリケーション設定`を作成します。
![multi-region-efs-4.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/125a732d-64a7-57a4-5ffd-43ebaa16fdbb.png)

#### (2).レプリケーション状態確認
次にレプリケーションの状態をメトリクスなどから確認します。
レプリケーションの同期履歴は以下で確認できます。

* 最終同期 (EFSの詳細タブから確認)
![efs-replication.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/5df08862-9450-d8a4-40f5-95b3d8e06e84.png)

* TimeSinceLastSync (Cloud Watch Metricsにて確認)
![efs-metrics-mini.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/011cdd31-a562-d035-0c0a-d1b4a3837ff7.png)

なおこれは`「レプリケーションの最終同期タイミング」`を表しており、実際にレプリケーションが発生しているかではありません。
厳密にEFSが利用されていないか確認したい場合は、`実際にEFSを参照する`、
またはメトリクスにて`DataReadIOBytes`、`DataWriteIOBytes`を参照するのが確実です。

* [EFS の CloudWatch Amazon メトリクス](https://docs.aws.amazon.com/ja_jp/efs/latest/ug/efs-metrics.html)

#### (3).レプリケーション設定削除
レプリケーションの状況を見て、データの読み込み/書き込みなど更新が発生していないことを確認できたら、
`Region B → Region A のレプリケーション設定`を削除します。
![multi-region-efs-5.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/c1da3027-a99e-0552-b8d7-50ed191679d9.png)

#### (4).EFSが利用できることを確認
レプリケーション設定の削除が完了した段階で Region A のEFSが書き込み可能となるので、
EC2から利用可能なことを確認します。
![multi-region-efs-6.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/1fe06726-4ad5-501b-d322-70a4c7e28232.png)

#### (5).レプリケーション設定作成
最後に`Region A → Region B のレプリケーション設定`を作成します。
![multi-region-efs-7.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/5a1bc186-abd6-5266-ea4b-933b51b2433d.png)

<a id="#Chapter2"></a>

# Terraformでのやり方
Terraformの場合もやることは同様であり、
`aws_efs_replication_configuration`がレプリケーション設定に該当するリソースなので、こちらを作成/削除します。

* [aws_efs_replication_configuration](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/efs_replication_configuration)

今回は例として、冒頭でも紹介した以下リポジトリを利用します。
* [multi-region-efs](https://github.com/Lamaglama39/terraform-for-aws/tree/main/app/multi-region-efs)

### ➀フェイルオーバー

#### (1).レプリケーション設定削除
該当の箇所をコメントアウトしてterraform applyし、
`Region A → Region B のレプリケーション設定`を削除します。
筆者の環境では削除完了まで8分程度かかりました。

```terraform:examples/main.tf
# module "efs_replication" {
#   source = "../modules/efs_replication"
# 
#   source_file_system_id = module.efs.efs.id
#   replication_file_system_id = module.efs_secondary.efs.id
#   replication_region = local.secondary_region
# }
```

#### (2).EFSが利用できることを確認
[こちら](#EFS-フェイルオーバー/フェイルバックの流れ)と同様

### ➁フェイルバック

#### (1).レプリケーション設定作成
該当の箇所をアンコメントしてterraform applyし、
`Region B → Region A のレプリケーション設定`を作成します。
筆者の環境では作成完了まで15分程度かかりました。

```terraform:examples/main.tf
module "efs_replication_secondary" {
  source = "../modules/efs_replication"

  providers       = { aws = aws.replica }
  source_file_system_id = module.efs_secondary.efs.id
  replication_file_system_id = module.efs.efs.id
  replication_region = local.primary_region
}
```

#### (2).レプリケーション状態確認
[こちら](#EFS-フェイルオーバー/フェイルバックの流れ)と同様

#### (3).レプリケーション設定削除
該当の箇所をコメントアウトしてterraform applyし、
`Region B → Region A のレプリケーション設定`を削除します。
筆者の環境では削除完了まで8分程度かかりました。

```terraform:examples/main.tf
# module "efs_replication_secondary" {
#   source = "../modules/efs_replication"
# 
#   providers       = { aws = aws.replica }
#   source_file_system_id = module.efs_secondary.efs.id
#   replication_file_system_id = module.efs.efs.id
#   replication_region = local.primary_region
# }
```

#### (4).EFSが利用できることを確認
[こちら](#EFS-フェイルオーバー/フェイルバックの流れ)と同様

#### (5).レプリケーション設定作成
該当の箇所をアンコメントしてterraform applyし、
`Region A → Region B のレプリケーション設定`を作成します。
筆者の環境では作成完了まで15分程度かかりました。

```terraform:examples/main.tf
module "efs_replication" {
  source = "../modules/efs_replication"

  source_file_system_id = module.efs.efs.id
  replication_file_system_id = module.efs_secondary.efs.id
  replication_region = local.secondary_region
}
```

<a id="#Chapter3"></a>

# まとめと所感
* 切替そのものはめちゃシンプルで手間が少ないのはうれしいポイント
* とはいえ切替完了までにちょっと時間がかかるので、余裕をもった計画が必要
* `RTO 15分以下`、`SLA 99.5%`みたいな超タイトな要件だと、正直厳しめかも…
* レプリケーションの進捗をダイレクトに見られるメトリクスがないのは、ちょっとモヤッとする

総じて便利で手軽ではありますが、即時性や管理面で細かい不満は残る感じでしょうか。
実運用で導入するなら手順の検証やダウンタイム計画、メトリクス監視などの工夫はしっかりやっておいた方が安心だと思います。
