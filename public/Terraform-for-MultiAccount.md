---
title: Terraformで始めるAWSマルチアカウント構築
tags:
  - AWS
  - Terraform
private: false
updated_at: '2024-01-22T23:07:22+09:00'
id: d12382559a48a99c9623
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに
<!-- 発端や概要を記載 -->
皆様はAWSでマルチアカウント構築に関わったことはありますでしょうか？

各アカウントの担当が同じであればマシですが、
アカウントごとに別担当なことが大半であり、コミュニケーションコストが爆増して苦労するのはあるあるですよね。

そんなときはプライベートでTerraformを利用してもろもろ検証しましょう！
またマルチアカウント構築をするにあたりいくつか方法があるので、本記事ではそれぞれ紹介します。

:::note warn
本記事ではAdministratorAccess権限を利用していますが、
IAM権限も含めて検証したい場合は各自適切なIAMポリシーを作成してください。
:::


<!-- 各チャプター -->
<a id="#Chapter1"></a>

# 1.マルチアカウント構築の設定

Terraformでマルチアカウントでリソースを定義するためには、
大まかに以下の二通りがあります。

- プロファイルのみ利用
- プロファイルとAssumeRoleを合わせて利用

## 1.1 プロファイルのみ利用
AWS CLIプロファイルで各アカウントの認証情報を管理する方法です。
Terraformを実行する端末のプロファイルで設定が完結するため、
簡易的に設定したい場合に有用な方法です。

![terraform_profile.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/fdbc2252-5ae5-b100-4867-7062a72b9c03.png)

### ① IAMユーザー作成
まずは各アカウントにてTerraform実行用にIAMユーザーを作成します。
```bash:IAMユーザー
# IAMユーザー作成
aws iam create-user \
  --user-name <アカウント① IAMユーザー名>

# IAMポリシーアタッチ
aws iam attach-user-policy \
  --user-name <アカウント① IAMユーザー名> \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

次に作成したIAMユーザーを指定してアクセスキーを作成します。
実行結果として出力される「AccessKeyId」、「SecretAccessKey」をコピーしておいてください。
```bash:アクセスキー
# アクセスキー作成
aws iam create-access-key \
  --user-name <アカウント① IAMユーザー名>

# 出力
{
    "AccessKey": {
        "UserName": "<アカウント① IAMユーザー名>",
        "AccessKeyId": "<アクセスキーID>",
        "Status": "Active",
        "SecretAccessKey": "<シークレットアクセスキーID>",
        "CreateDate": "2024-01-01T00:00:00+00:00"
    }
}
```

### ② プロファイル設定
次にterraformを実行する端末のプロファイル設定を行います。
各アカウントで作成したアクセスキーIDを設定ファイルに追記します。
```text:~/.aws/credentials
[account1]
aws_access_key_id = <アクセスキーID>
aws_secret_access_key = <シークレットアクセスキーID>

[account2]
aws_access_key_id = <アクセスキーID>
aws_secret_access_key = <シークレットアクセスキーID>
```

### ③ Terraform設定
プロファイル設定で記載した値を、
Terraformのproviderブロック内のprofile属性で指定します。
```terraform:Terraform設定
# アカウント1用のプロバイダー
provider "aws" {
  alias   = "account1"
  region  = "ap-northeast-1"
  profile = "account1"
}

# アカウント2用のプロバイダー
provider "aws" {
  alias   = "account2"
  region  = "ap-northeast-1"
  profile = "account2"
}
```

### ④ Terraformリソース定義
あとはTerraformでリソースを定義する際にproviderを指定してあげれば、
アカウントの指定ができます。
以下は例として、アカウント1、アカウント2でそれぞれS3バケットを作成しています。
```terraform:リソース定義例
resource "aws_s3_bucket" "account1" {
  bucket   = "s3-account1"
  provider = aws.account1
  force_destroy = true
}

resource "aws_s3_bucket" "account2" {
  bucket   = "s3-account2"
  provider = aws.account2
  force_destroy = true
}
```

## 1.2 プロファイルとAssumeRoleを合わせて利用
管理用アカウントのIAMユーザーを利用して、各アカウントのIAMロールへAssumeRoleする方法です。

前述の「1.1 プロファイルのみ利用」と比較すると、
アカウント単位のIAMユーザー(アクセスキー)の払い出しが不要なため、比較的セキュアに設定ができます。
また管理用アカウントを用意することで権限を集約できるため、アカウント管理の観点でも有用です。

![terraform-assume-role.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/3427041e-e985-cbc5-e3b3-981543ea6a2e.png)

### ①IAMユーザー作成
まずは管理用アカウントにてTerraform実行用にIAMユーザーを作成します。
```bash:IAMユーザー
# IAMユーザー作成
aws iam create-user \
  --user-name <管理用アカウント IAMユーザー名>

# IAMポリシーアタッチ
aws iam attach-user-policy \
  --user-name <管理用アカウント IAMユーザー名> \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

次に管理用アカウントのIAMユーザーを指定して、アクセスキーを作成します。
実行結果として出力される「AccessKeyId」、「SecretAccessKey」をコピーしておいてください。

```bash:アクセスキー作成
# アクセスキーを作成
aws iam create-access-key \
  --user-name <管理用アカウント IAMユーザー名>

# 出力
{
    "AccessKey": {
        "UserName": "userName",
        "AccessKeyId": "<アクセスキーID>",
        "Status": "Active",
        "SecretAccessKey": "<シークレットアクセスキーID>",
        "CreateDate": "2024-01-01T00:00:00+00:00"
    }
}
```

### ② プロファイル設定
次にterraformを実行する端末のプロファイル設定を行います。
管理用アカウントのIAMユーザーのアクセスキーIDを設定ファイルに追記します。
```text:~/.aws/credentials
[account1]
aws_access_key_id = <アクセスキーID>
aws_secret_access_key = <シークレットアクセスキーID>
```

### ③ IAMロール設定
次に管理用アカウントからAssumeRoleするためのIAMロールを、
各アカウントで作成します。
```bash:IAMロール作成
# IAMロール作成
aws iam create-role \
  --role-name <IAMロール名> \
  --assume-role-policy-document \
'{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Principal": {
                "AWS": "<管理用アカウントのID>"
            }
        }
    ]
}'

# IAMポリシーアタッチ
aws iam attach-role-policy \
  --role-name <IAMロール名> \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

### ④ Terraform設定
次に作成した各アカウントのIAMロールのARNを、
Terraformのproviderブロック内のassume_role属性に設定します。

```terraform:Terraform設定
# アカウント1用のプロバイダー
provider "aws" {
  alias   = "account1"
  region  = "ap-northeast-1"
  assume_role {
    role_arn = "<アカウント1 IAMロールのARN>"
  }
}

# アカウント2用のプロバイダー
provider "aws" {
  alias   = "account2"
  region  = "ap-northeast-1"
  assume_role {
    role_arn = "<アカウント2 IAMロールのARN>"
  }
}
```

### ⑤ Terraformリソース定義
あとはTerraformでリソースを定義する際にproviderを指定してあげれば、
アカウントの指定ができます。
以下は例として、アカウント1、アカウント2でそれぞれS3バケットを作成しています。
```terraform:リソース定義例
resource "aws_s3_bucket" "account1" {
  bucket   = "s3-account1"
  provider = aws.account1
  force_destroy = true
}

resource "aws_s3_bucket" "account2" {
  bucket   = "s3-account2"
  provider = aws.account2
  force_destroy = true
}
```


<a id="#Chapter3"></a>

# 2.効率良くリソースを定義する方法

次にリソースを効率よく定義する方法についてです。
本項目の内容を意識せずともマルチアカウント構築は可能ですが、
重複の多いコードを生成しないためにも、参考にしていただけると幸いです。

## 2.1 条件分岐、繰り返し処理を使用する
基礎的な構文ですが作成リソースが複数存在する場合は有用です。
以下はそれぞれの利用例です。

### ① 三項演算子
Terraformでの条件分岐は基本的に三項演算子で記述します。

* [Terraform 三項演算子](https://developer.hashicorp.com/terraform/language/expressions/conditionals)

定義された変数を判別して、作成するリソースのパラメータを変更する事ができます。
以下は変数envがprdの場合は「t3.large」、それ以外の場合は「t3.micro」のインスタンスタイプのEC2を作成します。

```terraform:三項演算子
variable "env" {
  type    = string
  default = "prd"
}

resource "aws_instance" "server" {
  # var.envの値を判別して、インスタンスタイプを指定
  instance_type = var.env == "prd" ? "t3.large" : "t3.micro"
  ami           = "ami-XXXXXXXXXX"
  subnet_id     = "subnet-XXXXXXXX"
}
```

### ② count
同じリソースを繰り返し作成できます、特定の数のリソースを作成する際に便利です。

* [Terraform count](https://developer.hashicorp.com/terraform/language/meta-arguments/count)

以下はEC2を5台作成し、それぞれのNameタグにindex+1の数を指定しています。
```terraform:count
resource "aws_instance" "server" {
  # 作成するリソースの数
  count = 5
  instance_type = "t3.micro"
  ami           = "ami-XXXXXXXXXX"
  subnet_id     = "subnet-XXXXXXXX"

  tags = {
    # count.indexで0から始まる値を設定
    Name = "server-${count.index + 1}"
  }
}
```

### ③ for_each
for_eachを利用することで、複数のリソースを異なる値で作成できます。
またfor_eachへ渡す値で使用できるのは、コレクション型（mapまたはset）です。

* [Terraform for_each](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each)

以下はmapを使用したIAMユーザーの作成例です。
each.keyでは、var.usersの各キー（person1、person2、person3）を指定し、
each.valueでは、var.usersの各値（admin、developer、sysops）を指定しています。
```terraform:for_each
variable "users" {
  default = {
    person1  = "admin"
    person2  = "developer"
    person3  = "sysops"
  }
}

resource "aws_iam_user" "user" {
  for_each = var.users

  name = each.key
  tags = {
    Name = "iam-user-${each.value}"
  }
}
```

## 2.2 モジュールを使用する
モジュールを使用することでコードの重複を避け、効率的に管理することができます。
余程小規模な開発ではければ利用することが推奨されます。

* [Terraform モジュール](https://developer.hashicorp.com/terraform/language/modules)

以下は独自にmoduleを作成して利用している例です。

```text:ディレクトリ構成
.
├── main.tf
└── modules
    └── EC2
        └── main.tf
```
```terraform:モジュールの定義 (./modules/EC2/main.tf)
variable "instance_type" {
  type    = string
  default = "t3.micro"
}
variable "server_name" {
  type    = string
  default = "ec2"
}

resource "aws_instance" "server" {
  ami           = "ami-XXXXXXXXXXXX"
  instance_type = var.instance_type
  tags = {
    Name = "${var.server_name}-server"
  }
}
```
```terraform:モジュールの呼び出し (./main.tf)
module "app_server" {
  source        = "./modules/EC2"
  instance_type = "t3.large"
  server_name   = "app"
}
```


または公開されているモジュールを利用することもできます。
大抵のAWSのリソースはモジュールが公開されているため、自作する前に覗いてみるのもよいかと思います。

* [Terraform Registry](https://registry.terraform.io/browse/modules)

## 2.3 Workspacesを使用する
複数の環境（開発、ステージング、本番など）で複数のAWSアカウントを使用する場合、
Workspacesを使用して環境ごとに独立した状態を保持することが有用です。

* [Terraform Workspaces](https://developer.hashicorp.com/terraform/cli/workspaces)

具体的な利用方法は公式ドキュメントを参照したほうが分かりやすいと思うので、
ここではworkspaceを利用する際のコマンド例のみ記載します。
```bash:Workspacesの利用
#workspaceを作成
terraform workspace new <ワークスペース名>

#workspaceを切替
terraform workspace select <ワークスペース名>

#workspaceを削除
terraform workspace delete <ワークスペース名>

#workspaceの一覧を表示
terraform workspace list

#現在のworkspaceを表示
terraform workspace show
```

<a id="#Chapter3"></a>

# 3.最後に
最後になりますが、
`terraform destory`忘れだけは本当に気を付けてください。
~~(リソースによっては利用料金が結構えげつないことになるので...。)~~
