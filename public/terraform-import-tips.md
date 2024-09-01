---
title: 正体不明なTerraform管理外リソースのトリセツ。
tags:
  - IaC
  - 運用
  - Terraform
  - 保守
private: false
updated_at: '2024-09-01T23:12:59+09:00'
id: 074410e7fba345af5174
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- 発端や概要を記載 -->
# はじめに

業務でTerraformを使っていると、「Terraform管理外のリソースたち」に遭遇することがあります。
`PoCで作成したものがそのまま残っていたり`、`やむを得ず手動で作成したリソースが放置されたり`…。
`Terraformで管理したい気持ちはあっても、気づけば放置されている可哀そうなサーバだったり`…。
このような正体不明なリソースは、
リリース済みのシステムにアサインされると意外によく見かけます。

こういったリソースに対して運用改善やコスト削減といった名目で発生するタスクは、
プロジェクトの新規参画者に割り振られる事が往々にしてあります。
そして全く知らないシステムのリソースをimportする羽目になり困惑することもしばしば...。
~~勉強にはなるので、悪い事ばかりではないですが...。~~

そんな困った状況に陥ったTerraformユーザーに向けて、
Terraform管理外のリソースをどのように扱うべきかについて解説します。

:::note warn
本記事はあくまでリソースの大まかな調査方法や、一般的な対処方法をまとめた記事です。
最終的な判断は必ず現場で確認してください。
:::

<!-- 各チャプター -->
# 事前確認

初めに調査対象のリソースの要否と用途を確認しましょう。
要否レベルで不明なリソースはまずないと思いますが、
`設計書などのドキュメントでの確認`、`先輩たちへの聞き込み調査`、`構築時のログなどでの履歴確認`…etc…。
どんな方法でもいいので、最低限以下の事項を確認してリソースの正体を解き明かしましょう。

```text:(1).リソースの重要性と利用状況確認
リソースは現在のシステムで必要か？

 ➤ YES（例：重要な役割を持っている、システムの一部として利用されている）
 ➤ NO （例：不要、または役割が不明）
```
```text:(2).リソースの管理状況の確認
リソースは現在どのように管理されているか？

 ➤ 手動で管理     （例：手動で作成した、または外部ツールから管理されている）
 ➤ 他ツールで管理 （例：別のIacツール、カスタムスクリプトなど）
```
```text:(3).リソースの管理方法の決定
リソースをTerraformで管理する必要があるか？

 ➤ YES（例：Terraformで統一的に管理したい、運用の効率化を図りたい）
 ➤ NO （例：現在の管理方法で問題ない、または運用に影響がない）
```

最終的な判断は現場によりますが、
上記の確認を踏まえて、リソースの処遇は大まかに以下の3パターンに分かれるでしょう。

* [1.リソースを削除する](#1リソースを削除する)
* [2.DataSourcesとして定義する](#2datasourcesとして定義する)
* [3.Terraform管理下にする](#3terraform管理下にする)

<a id="#Chapter1"></a>

## 1.リソースを削除する

リソースに以下のような経緯がある場合、削除することが多いでしょう。

* PoC検証時の残骸
* 障害復旧などで一時的に必要になったリソースの残骸
* 移行完了後の旧リソース

言うまでもないですが、`削除間違い`にだけは気をつけてください。
また`他のリソースとの依存関係`の有無をよく確認した上で削除しましょう。

なお大抵の場合はCLIツールが用意されているので、確実性を上げるためにも利用を検討してください。

<a id="#Chapter2"></a>

## 2.DataSourcesとして定義する

リソースが以下のような用途の場合、DataSources(データソース)として定義することが多いでしょう。
ただ基本的に構築段階で対応しているべき内容なので、このパターンはまずないと思います。

* パスワード、SSH鍵などのシークレット情報
* 共通の設定を維持するために、既存のリソースの情報を参照
* 既存の設定や状態を確認のために情報を取得する

以下は一例ですが、
AWS CLI でシークレットを作成してdataブロックにて指定し、
resourceブロックにてRDSのAdminユーザー名/パスワードとして参照しています。

```console:シークレット作成
$ aws secretsmanager create-secret \
    --name db-secret \
    --secret-string '{"username":"dbuser","password":"dbpassword"}'
```

```terraform:main.tf
provider "aws" {
  region = "ap-northeast-1"
}

data "aws_secretsmanager_secret" "main" {
  name = "db-secret"
}

data "aws_secretsmanager_secret_version" "main" {
  secret_id = data.aws_secretsmanager_secret.main.id
}

locals {
  secret_data = jsondecode(data.aws_secretsmanager_secret_version.main.secret_string)
  db_username = local.secret_data.username
  db_password = local.secret_data.password
}

resource "aws_db_instance" "main" {
  # シークレットマネージャーの値を参照
  username          = local.db_username
  password          = local.db_password

  identifier        = "postgres-rds-instance"
  instance_class    = "db.t3.micro"
  engine            = "postgres"
  engine_version    = "16.4"
  allocated_storage = 20
  db_name           = "testdatabase"
  final_snapshot_identifier = false
}
```

この状態でterraform applyすることでDataSourcesとしてtfstateに追加され、
以下コマンドで一覧を確認できます。

```console:DataSources 確認
$ terraform state list
data.aws_secretsmanager_secret.main
data.aws_secretsmanager_secret_version.main
aws_db_instance.main
```

なお逆にDataSourcesを削除する際に、
dataブロックを削除してterraform applyしてもtfstateから削除されないことがあります。

そういった場合は`terraform state rm`で指定することでtfstateから削除できます。
ただし削除前に必ず他のresourceから参照されていないことを確認してください。
```console:DataSources 削除例
$ terraform state rm data.aws_secretsmanager_secret.main
```

DataSourcesの定義方法は公式ドキュメントに記載があるので、
詳細は以下で確認してください。

* [Data Sources](https://developer.hashicorp.com/terraform/language/data-sources)
* [Terraform providers](https://registry.terraform.io/browse/providers)


<a id="#Chapter3"></a>

## 3.Terraform管理下にする

以下のようなリソースの場合、importしてTerraform管理下にすることになるでしょう。
何だかんだこのパターンが一番遭遇率が高いと思います。

* 初期開発時の手動作成リソース
* terraform導入前に作成したリソース

またいくつか方法があるので、それぞれのメリット/デメリットを合わせて解説します。
例として、AWS CLIで作成したEC2インスタンスをimportする流れを記載します。

* [1.importコマンド](#1importコマンド)
* [2.importブロック](#2importブロック)
* [3.terraformer](#3terraformer)

### (1).importコマンド

importコマンドは指定したリソースの定義情報をtfstateに追加します。
初期のTerraformから実装されている機能のためバージョン問わず利用できますが、
事前の差分確認ができず、またresourceブロックの手動修正が必要となるため可能であれば後述のimportブロックの利用を推奨します。

* [importコマンド 公式ドキュメント](https://developer.hashicorp.com/terraform/cli/import)

```text:メリット
・Terraform標準の方法
Terraformに組み込まれている公式の手段であり、追加のツールが不要

・手動での精密な制御
インポートするリソースの詳細を指定でき、ターゲットを明確に制御できる
```
```text:デメリット
・実行前に差分確認ができない
コマンド実行前にtfstateの差分が確認できない

・resourceブロックの自動生成機能がない
resourceブロックの生成機能はないため、コマンド実行後は差分が消えるまで修正し続ける必要がある

・複数リソースの一括インポートに不向き
リソースを一つずつインポートするため、インポート対象が多い場合はコマンドを個別に実行する必要があり手間がかかる
```

**① インポート用リソース作成**
AWS CLI でEC2インスタンスを作成します。

```console
# AmazonLinux2023の最新AMIを取得
$ TARGET_AMI=$(aws ssm get-parameters \
    --names "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64" \
    --region ap-northeast-1 \
    --query "Parameters[*].Value" \
    --output text
    )

$ aws ec2 run-instances \
    --image-id $TARGET_AMI \
    --instance-type t3.micro
```

**② resourceブロック作成**
インポート対象リソース用のresourceブロックを定義します、
なおimportコマンド後に手動修正するため、この時点では設定値の記載は不要です。

```terraform:main.tf
resource "aws_instance" "main" {
}
```

**③ importコマンド実行**

importコマンドは以下の引数を指定します。

```console
$ terraform import [resource type].[resource name] [import対象リソースのID]
```

EC2の場合はインスタンスIDを指定します。
importコマンド実行後に`Import successful!`と出力されればtfstateにインポートできています。
$ terraform import aws_instance.main i-123456789123456789
```
```console:実行結果
Import successful!

The resources that were imported are shown above. These resources are now in
your Terraform state and will henceforth be managed by Terraform.
```

tfstateを見るとインポート対象が追加されていることが確認できます。
```json:tfstate 抜粋
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "main",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
```

**④ resourceブロック修正**
terraform planで差分を確認してresourceブロックを修正します。
以下の実行結果から、aws_instanceは`ami`、`instance_type`が必須パラメータのため定義する必要があることがわかります。

```cosole
$ terraform plan
╷
│ Error: Missing required argument
│ 
│   with aws_instance.main,
│   on main.tf line 5, in resource "aws_instance" "main":
│    5: resource "aws_instance" "main" {
│ 
│ "launch_template": one of `ami,instance_type,launch_template` must be specified
╵
╷
│ Error: Missing required argument
│ 
│   with aws_instance.main,
│   on main.tf line 5, in resource "aws_instance" "main":
│    5: resource "aws_instance" "main" {
│ 
│ "ami": one of `ami,launch_template` must be specified
╵
╷
│ Error: Missing required argument
│ 
│   with aws_instance.main,
│   on main.tf line 5, in resource "aws_instance" "main":
│    5: resource "aws_instance" "main" {
│ 
│ "instance_type": one of `instance_type,launch_template` must be specified
╵
```

以下のようにresourceブロックに定義を追加します。

```terraform
resource "aws_instance" "main" {
  ami           = "ami-00c79d83cf718a893"
  instance_type = "t3.micro"
}
```

再度差分を確認をして、`No changes.`と出力されればインポート完了です。
ここで差分が発生する場合は、差分が無くなるまでresourceブロックを修正します。

```console
$ terraform plan
aws_instance.main: Refreshing state... [id=i-00c5ef73e12b18394]
```
```console
No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and found no differences, so no changes are needed.
```


### (2).importブロック

importブロックはTerraform v1.5.0 で実装された機能であり、
importコマンドに比べメリットが多いため利用できる場合はこちらをおすすめします。

* [importブロック 公式ドキュメント](https://developer.hashicorp.com/terraform/language/import)

```text:メリット
・事前の差分確認
事前にどのようなリソースがインポートされるか差分を確認できる

・複数のリソースのインポート
importブロックにて複数のリソースを定義できるため、一括でインポートができる

・Terraform標準の方法
Terraformに組み込まれている公式の手段であり、追加のツールが不要
```
```text:デメリット
・利用できるバージョンの制約
Terraform v1.5.0 から利用可能であるため、プロジェクトによっては利用できない場合がある
```

importコマンドの場合はリソースの定義情報がtfstate上にしか追加されないため、resourceブロックは手動で修正する必要がありますが、
importブロックはresourceブロックも自動で生成できます。
またimportコマンドはリソースごとに個別にインポートする必要がありますが、importブロックは複数のリソースをまとめてインポートすることができます。

なおimportブロックの使用方法として以下の二通りあるので、それぞれ解説します。

[1.resourceブロックを自動で生成する](#1resourceブロックを自動で生成する)
[2.resourceブロックを事前に作成する](#2resourceブロックを事前に作成する)

#### 1.resourceブロックを自動で生成する

**① インポート用リソース作成**
importコマンドの例と同様に、AWS CLIでEC2インスタンスを作成します。
また複数リソースのインポートの例として、二つ作成します。

**② importブロック作成**
importブロックは以下のようにインポートするリソースごとに定義します。
また`id`と`to`は必須となるため対象に合わせて記載してください、
値自体はimportコマンドの引数で指定したものと同様です。

```terraform:import.tf
import {
  id = "i-123456789123456789"
  to = aws_instance.web
}

import {
  id = "i-987654321987654321"
  to = aws_instance.ap
}
```

なおこの際に必須になる引数の記載例は公式ドキュメントで確認できます。
EC2インスタンスの場合は以下に記載があります。

* [aws_instance import](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance#import)

**③ リソース定義 インポート**
terraform planに`-generate-config-out=[ファイル名].tf`という引数を付けて実行することで、
resourceブロックが記述されたファイルが生成されます。

```console
$ terraform plan -generate-config-out=main.tf
aws_instance.web: Preparing import... [id=i-123456789123456789]
aws_instance.ap: Preparing import... [id=i-987654321987654321]
aws_instance.web: Refreshing state... [id=i-123456789123456789]
aws_instance.ap: Refreshing state... [id=i-987654321987654321]
```

なおファイル生成後に自動的にterraform planが実施されます。
EC2インスタンスの場合はresourceブロック内の`ipv6_addresses`と`ipv6_address_count`が競合してしまうためエラーが発生します。

```
Planning failed. Terraform encountered an error while generating this plan.

╷
│ Warning: Config generation is experimental
│ 
│ Generating configuration during import is currently experimental, and the generated configuration format may change in future versions.
╵
╷
│ Error: Conflicting configuration arguments
│ 
│   with aws_instance.ap,
│   on main.tf line 14:
│   (source code not available)
│ 
│ "ipv6_address_count": conflicts with ipv6_addresses
╵
╷
│ Error: Conflicting configuration arguments
│ 
│   with aws_instance.ap,
│   on main.tf line 15:
│   (source code not available)
│ 
│ "ipv6_addresses": conflicts with ipv6_address_count
╵
```

そのためどちらかをコメントアウトします、
ipv6をそもそも利用していないようであれば両方コメントアウトして問題ありません。

```terraform:main.tf
#  ipv6_address_count = 0
#  ipv6_addresses     = []
```

その後再度terraform planを実行し、`import`以外の差分が発生しないことを確認します。

```console
$ terraform plan
```
```console:実行結果
Plan: 2 to import, 0 to add, 0 to change, 0 to destroy.
```

**④ tfstate インポート**
terraform applyコマンドを実行してtfstateにリソースの定義情報を追加します。

```console
$ terraform apply
aws_instance.web: Importing... [id=i-123456789123456789]
aws_instance.web: Import complete [id=i-123456789123456789]
aws_instance.ap: Importing... [id=i-987654321987654321]
aws_instance.ap: Import complete [id=i-987654321987654321]
```
```console:実行結果
Apply complete! Resources: 2 imported, 0 added, 0 changed, 0 destroyed.
```

コマンド実行後、tfstateにリソースの定義情報が追加されていることが確認できます。
またこの段階でimportブロックは不要となるため削除して問題ありません。

```diff:main.tf
- import {
-   id = "i-123456789123456789"
-   to = aws_instance.web
- }

- import {
-   id = "i-987654321987654321"
-   to = aws_instance.ap
- }
```

#### 2.resourceブロックを事前に作成する
importブロック内の`to`に指定したresourceブロックが記述されたtfファイルが存在する場合、
tfファイルの生成を行わず、既存のresourceブロックと`id`で指定したリソースを紐づけることが出来ます。

**① インポート用リソース作成**
importコマンドの例と同様に、AWS CLIでEC2インスタンスを作成します。
また複数リソースのインポートの例として、二つ作成します。

**② importブロック、resourceブロックの作成**
importブロック、resourceブロックをそれぞれ以下のように定義します。

```terraform:import.tf
import {
  id = "i-123456789123456789"
  to = aws_instance.web
}

import {
  id = "i-987654321987654321"
  to = aws_instance.ap
}
```
```terraform:main.tf
resource "aws_instance" "web" {
  ami           = "ami-00c79d83cf718a893"
  instance_type = "t3.micro"
}

resource "aws_instance" "ap" {
  ami           = "ami-00c79d83cf718a893"
  instance_type = "t3.micro"
}
```

**③ インポートの実行**
terraform applyコマンドを実行してtfstateにリソースの定義情報を追加します。

```console
$ terraform apply
aws_instance.web: Importing... [id=i-123456789123456789]
aws_instance.web: Import complete [id=i-123456789123456789]
aws_instance.ap: Importing... [id=i-987654321987654321]
aws_instance.ap: Import complete [id=i-987654321987654321]
```
```console:実行結果
Apply complete! Resources: 2 imported, 0 added, 0 changed, 0 destroyed.
```

コマンド実行後、tfstateにリソースの定義情報が追加されていることが確認できます。
またこの段階でimportブロックは不要となるため削除して問題ありません。

```diff:main.tf
- import {
-   id = "i-123456789123456789"
-   to = aws_instance.web
- }

- import {
-   id = "i-987654321987654321"
-   to = aws_instance.ap
- }
```


### 3.terraformer
terraformerはGoogle Cloud PlatformがOSSとして提供しているツールであり、
指定した環境内のリソースを読み取って定義情報をtfファイルとして出力します。
アカウント内の広い範囲をインポートできるため、手動で構築したシステムを丸ごとterraform管理化したい場合などに有用です。

```text:メリット
・tfファイルの自動生成
tfファイルを自動で生成できるため、手動で設定を書く手間が省ける

・リソースの一括インポート
複数のリソースを一度にインポートできるため、大規模なインフラの移行や管理が効率化される

```
```text:デメリット
・インポート対象の指定
環境内のリソースを一括でインポートするため、指定を誤るとデフォルトのリソースなどもインポートされてしまう場合がある

・外部ツールの依存
Terraform本体ではなく外部ツールを使用するため、ツールのバージョンや互換性に注意が必要
```

* [terraformer](https://github.com/GoogleCloudPlatform/terraformer)


**① terraformer インストール**
以下を参照して、実行環境に合わせてインストールしてください。
* [インストール方法](https://github.com/GoogleCloudPlatform/terraformer?tab=readme-ov-file#installation:~:text=pattern%20parameters%20together.-,Installation,-Both%20Terraformer%20and)

個人的にはmiseでインストールするのが楽なのでおすすめします。
```console
$ mise use terraformer@latest
```

**② terraform init**
プロバイダー情報を記載したtfファイルを作成し、terraform initで初期化します。

```terraform:provider.tf
provider "aws" {
  region = "ap-northeast-1"
}
```
```console
$ terraform init
```

**③ terraformer import**
terraformer importでtfファイル、tfstateを生成します。
コマンドは以下のように引数を指定します。

```console
$ terraformer import [プロバイダー] --resources=[インポート対象リソース]
```

また`--resources=`にて指定できるリソースは以下コマンドで確認できます。
```console
$ terraformer import [プロバイダー] list
```
```console:awsの場合
$ terraformer import aws list
```

例として環境内のEC2インスタンス、VPC、サブネットをすべて指定する場合は以下の通りです。

```console
$ terraformer import aws --resources=ec2_instance,vpc,subnet
```
```console:実行結果
2024/09/01 22:08:32 aws importing default region
2024/09/01 22:08:33 aws importing... ec2_instance
2024/09/01 22:08:35 aws done importing ec2_instance
2024/09/01 22:08:35 aws importing... vpc
2024/09/01 22:08:35 aws done importing vpc
2024/09/01 22:08:35 aws importing... subnet
2024/09/01 22:08:35 aws done importing subnet
2024/09/01 22:08:35 Number of resources for service ec2_instance: 2
2024/09/01 22:08:35 Number of resources for service vpc: 1
2024/09/01 22:08:35 Number of resources for service subnet: 3
2024/09/01 22:08:35 Refreshing state... aws_subnet.tfer--subnet-XXXXXXXXXXXXXXXXX
2024/09/01 22:08:35 Refreshing state... aws_subnet.tfer--subnet-XXXXXXXXXXXXXXXXX
2024/09/01 22:08:35 Refreshing state... aws_instance.tfer--i-XXXXXXXXXXXXXXXXX
2024/09/01 22:08:35 Refreshing state... aws_instance.tfer--i-XXXXXXXXXXXXXXXXX
2024/09/01 22:08:35 Refreshing state... aws_subnet.tfer--subnet-XXXXXXXXXXXXXXXXX
2024/09/01 22:08:35 Refreshing state... aws_vpc.tfer--vpc-XXXXXXXXXXXXXXXXX
2024/09/01 22:08:37 Filtered number of resources for service vpc: 1
2024/09/01 22:08:37 Filtered number of resources for service subnet: 3
2024/09/01 22:08:37 Filtered number of resources for service ec2_instance: 2
2024/09/01 22:08:37 aws Connecting.... 
2024/09/01 22:08:37 aws save subnet
2024/09/01 22:08:37 aws save tfstate for subnet
2024/09/01 22:08:37 aws save ec2_instance
2024/09/01 22:08:37 aws save tfstate for ec2_instance
2024/09/01 22:08:37 aws save vpc
2024/09/01 22:08:37 aws save tfstate for vpc
```

コマンドの実行に成功するとgeneratedディレクトリが生成されます。
`generated`>`プロバイダー`>`リソース`という構造になっており、リソースの種別ごとにtfファイル、tfstateファイルが作成されます。

なお`--resources=`引数の指定によっては想定外のリソースがインポートされることがあるので、
具体的にどのリソースがインポートされたか各リソースのtfファイル、tfstateを確認することをおすすめします。

```console:コマンド実行後のディレクトリ構成
.
├── generated
│   └── aws
│       ├── ec2_instance
│       │   ├── instance.tf
│       │   ├── outputs.tf
│       │   ├── provider.tf
│       │   ├── terraform.tfstate
│       │   └── variables.tf
│       ├── subnet
│       │   ├── outputs.tf
│       │   ├── provider.tf
│       │   ├── subnet.tf
│       │   ├── terraform.tfstate
│       │   └── variables.tf
│       └── vpc
│           ├── outputs.tf
│           ├── provider.tf
│           ├── terraform.tfstate
│           └── vpc.tf
└── provider.tf
```

その他リソースの指定方法については、
公式ドキュメントに使用例が記載されているのでご確認ください。

* [terraformer](https://github.com/GoogleCloudPlatform/terraformer)


# 参考ドキュメント
本記事の作成に際して、以下のドキュメントを参照させていただきました。
この場を借りてお礼申し上げます。

* [Terraformのimportコマンドとimportブロックを試してみた](https://dev.classmethod.jp/articles/terraform-import-command-and-import-block/)
* [terraform importで数年やってきたがImport blockの良さに気づきました](https://zenn.dev/aeonpeople/articles/d63e84494d9e2c#%E3%82%B9%E3%83%86%E3%83%BC%E3%83%88%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%82%92%E5%A4%89%E6%9B%B4%E3%81%9B%E3%81%9A%E3%81%AB%E3%82%B3%E3%83%BC%E3%83%89%E3%81%A8%E5%AE%9F%E6%85%8B%E3%81%AE%E5%B7%AE%E5%88%86%E3%82%92%E7%A2%BA%E8%AA%8D%E3%81%A7%E3%81%8D%E3%82%8B)
* [Terraformのimportブロックの使い方について簡潔にまとめる & 使ってみる](https://zenn.dev/not75743/articles/e890cec01533c9)
* [Linux環境でTerraformerを使って既存AWSリソースをコード化してみる](https://zenn.dev/rescuenow/articles/7f8cbcfa14697b)
