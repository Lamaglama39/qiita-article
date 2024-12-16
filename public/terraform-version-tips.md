---
title: Terraformのバージョン管理と制限する方法について
tags:
  - IaC
  - Terraform
private: false
updated_at: '2024-12-16T12:18:27+09:00'
id: cbc2ba524e3f75e0f6b9
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- 発端や概要を記載 -->
# はじめに
Terraformは本体やproviderなどバージョン管理の対象が複数存在し、
バージョンが変わることで`コードが動作しなくなる可能性`や、`想定外のエラー`が発生することあります。
そのため業務でTerraformを利用していると、バージョン管理は避けては通れません。

なので適切なバージョン管理を行い、各環境やチームでの一貫性を保つことは非常に重要ですが、
`各担当者の知識のムラ`や`プロジェクトの様々な歴史的背景`により、不完全な管理となっていることも珍しくありません…。
```text:よくある事例
プロジェクトA: バージョン1.1.0のみ利用可能
プロジェクトB: バージョン1.5.0以上はすべて利用可能
プロジェクトC: バージョン1.8.0以上、1.9.0未満の範囲は利用可能
.
.
プロジェクトX: そもそもバージョンを制限してない
```

そこで本記事ではそんな困った場面に遭遇したTerraformユーザーに向けて、
`バージョン管理`と`バージョンを制限する方法`について解説します。

<!-- 各チャプター -->
<a id="#Chapter1"></a>

# Terraformのバージョン管理について
Terraform本体のバージョン管理は開発環境の一貫性を保つために不可欠ですが、
前述の通り`プロジェクトやチームごとにバージョンが異なる`のはよくあることで、頻繁にバージョンの切り替えが必要となります。

そのためもし特定のバージョンを直にインストールしているのであれば、
管理ツールを利用することを推奨します。

### 専用バージョン管理ツール（tfenv、tenv）
tfenvはTerraform専用のバージョン管理ツールで、
特定のプロジェクトやディレクトリに対応するTerraformのバージョンを簡単に切り替えられるのが特徴です。

後継としてtenvが提供されており、
OpenTofu、Terraform、Terragrunt、AtmosなどTerraform関連に特化した様々なツールのバージョン管理が可能です。
またGo言語で書き直されておりマルチプラットフォームで動くので、`tenvをオススメ`します。

* [tfenv](https://github.com/tfutils/tfenv)
* [tenv](https://github.com/tofuutils/tenv)

```text:特徴
・Terraformに特化した管理機能
・.terraform-version、または.tfswitchrcを使用した自動切り替え
```

### 汎用バージョン管理ツール (asdf、mise)
asdfやmiseは複数のプログラミング言語やツールを一元管理できる汎用ツールで、Terraformのバージョン管理にも対応しています。
特に他の言語やツールを利用している場合は、同時にバージョン管理できるので便利です。

またmiseはasdfのRust版のような立ち位置であり、
互換性があり軽快に動くので特にこだわりがなければ`miseをオススメ`します。

* [asdf](https://github.com/asdf-vm/asdf)
* [mise](https://github.com/jdx/mise)

```text:特徴
・複数ツールを一括管理可能
・様々なプラグインに対応しており拡張性が高い
・.tool-versionsファイルによるプロジェクトごとの設定
```

asdf、miseのコマンド一覧については以下記事にまとめているので、よければご参照ください。
* [asdf/mise コマンド比較](https://qiita.com/Lamaglama39/items/70a359b4f4c613eb2ee6)

<a id="#Chapter2"></a>

# バージョンを指定できる対象について
Terraformでバージョンを指定できる対象は、大きく分けて2種類あります。

### terraform
Terraform本体のバージョンを指定することで環境ごとの互換性を保ち、意図しない動作やエラーを防ぐことができます。
特にメジャーバージョンやマイナーバージョンをまたぐと、
`構文の変更`や`新機能の追加`などコードの修正が必要となる場合があるので`必ず指定`しましょう。

具体的には`terraformブロック`内の`required_version`ブロックで指定が可能で、
以下の例であれば`1.10.0`のバージョンの使用を強制します。

* [required_version](https://developer.hashicorp.com/terraform/language/terraform#terraform-required_version)

```terraform
terraform {
  required_version = "1.10.0"
}
```

### provider
providerは`特定のクラウドやサービスとのやり取りをするためのプラグイン`であり、
リソースの操作をTerraformから実行できるようにします。
providerが提供するリソースやデータソースの仕様はバージョンによって変わることがあるため、こちらも指定することが推奨されます。

具体的には`terraformブロック`内の`required_providers`ブロックで指定が可能です。
以下の例であればawsでは`5.80.0`のバージョンの使用を強制、
google cloudでは`6.10.0`のバージョンの使用を強制します。

* [required_providers](https://developer.hashicorp.com/terraform/language/terraform#terraform-required_providers)

```terraform
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "5.80.0"
    }
    google = {
      source  = "hashicorp/google"
      version = "6.10.0"
    }
  }
}
```


<a id="#Chapter3"></a>

# バージョンの指定方法について
Terraformではバージョンを柔軟に指定でき、用途に合った指定方法を選ぶ必要があります。
* [terraform-version-constraints](https://developer.hashicorp.com/terraform/tutorials/configuration-language/versions#terraform-version-constraints)

また`セマンティック バージョニング`に則っており、バージョンをX.Y.Zの形式で表し、
`Xはメジャーバージョン`、`Yはマイナーバージョン`、`Zはパッチバージョン`を意味します。
メジャーバージョン、マイナーバージョンはコードの修正を伴う大きな変更が発生する可能性があり、
パッチバージョンは`バグ修正`などが主なため、`基本的にコードへの影響は発生しません。`
* [versioning](https://developer.hashicorp.com/terraform/plugin/best-practices/versioning)

### バージョンX.Y.Zのみ
特定のバージョンのみを使用する場合に指定します。
パッチバージョンも含め、`完全に一致していることを強制`されるため最も厳しく制限できますが、`他のmoduleの指定と競合`することが多いためほとんど利用されません。

```terraform:1.5.0 のみを許可
terraform {
  required_version = "1.5.0"
}
```

### バージョンX.Y.Z以上
指定したバージョン以上を許可する場合に使用します。
許可される範囲が広くほとんど制限できないため、実際のプロダクトでは利用されない印象です。

```terraform:1.5.0 以上を許可
terraform {
  required_version = ">= 1.5.0"
}
```

### バージョンX.Y.Z、およびそのマイナーバージョン
指定したパッチバージョン範囲のみ許可します。

パッチバージョンであればアップデートしてもまず影響はないため、
実際のプロダクトでは一番よく利用する指定方法です。
何らかの特別な理由がなければ、`基本的にこの形式にしておけば問題ない`かと思います。

```terraform:1.5.0 以上 1.5.X 以下 を許可
terraform {
  required_version = "~> 1.5.0"
}
```

### バージョンX.X.X以上、バージョンY.Y.Y未満または以下
範囲指定で特定のバージョン区間を許可する場合に使用します。
他のmoduleとの兼ね合いで、`マイナーバージョンの範囲はある程度許可したい`場合などに利用します。

```terraform:1.5.0 以上 1.10.0 未満 を許可
terraform {
  required_version = ">= 1.5.0, < 1.10.0"
}
```
```terraform:1.5.0 以上 1.10.0 以下 を許可
terraform {
  required_version = ">= 1.5.0, <= 1.10.0"
}
```


<a id="#Chapter4"></a>

# バージョンを指定できる箇所について
Terraformではバージョンを指定できる箇所が複数存在し、それぞれ`優先度`や`適用範囲`が異なります。
今回は以下のディレクトリ構成を例に説明します。

```text:ディレクトリ構成
.
├── exsamples
│   └── main.tf
└── modules
    ├── s3
    │   └── main.tf
    └── vpc
        └── main.tf
```

なお`terraformブロック`、`providerブロック`は通常個別のファイル(provider.tf、versions.tf)などに切り出すことが多いですが、
本記事では他のリソースとまとめてmain.tfに記述します。

### root module
root module(terraform initを実行するディレクトリ)では、
`そのプロジェクト全体でのterraform、providerのバージョンを指定`できるため、`必ず記述`しましょう。
後述のmoduleでバージョンが指定されていない場合は、root moduleの指定が適用されます。

```terraform:exsamples/main.tf
terraform {
  required_version = "~> 1.10.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.80.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-1"
}

module "s3" {
  source = "../modules/s3"
}

module "vpc" {
  source = "../modules/vpc"
}
```

### module(ローカル)
ローカルモジュールでバージョンを指定することで、再利用する際に意図しない挙動を防止できます。
そのため`他プロジェクトと共通で利用するモジュール`などであれば、指定しておくのがbetterです。

```terraform:modules/s3/main.tf
terraform {
  required_version = "~> 1.10.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.80.0"
    }
  }
}

resource "aws_s3_bucket" "example" {
  bucket_prefix = "example-s3-bucket-"
}
```

### module(リモート)
Terraform RegistryやGithubリポジトリから取得するリモートモジュールの場合も、バージョンを指定することで意図しない更新を避けられます。
ただしこのバージョンは`リモートリポジトリのバージョン`のことであり、
`terraform、providerのバージョンはmodule側の指定に委ねられる`ので、こちらも指定するのがbetterです。

具体的には`sourceで参照先のリポジトリ`、`versionで参照先のリポジトリのタグ`を指定します。
以下であれば`terraform-aws-modules/vpc/awsの5.10.0`を指定しています。
```terraform:modules/vpc/main.tf
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.10.0"

  name = "example-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["ap-northeast-1a", "ap-northeast-1c", "ap-northeast-1d"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
}
```

参照しているリモートリポジトリ側のterraform、providerの指定は以下の通りです。
公式のリモートリポジトリは様々な利用者がいるため、互換性の問題が発生しないように`バージョン○○以上`といった緩い指定になっていることが多いです。
* [terraform-aws-vpc v5.10.0 versions.tf](https://github.com/terraform-aws-modules/terraform-aws-vpc/blob/v5.10.0/versions.tf)
```terraform:https://github.com/terraform-aws-modules/terraform-aws-vpc/blob/v5.10.0/versions.tf
terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.46"
    }
  }
}
```


<a id="#Chapter5"></a>

# 複数の箇所でバージョンを指定した場合の挙動について
required_versionやrequired_providersが`root module、複数のmoduleで指定されている`場合は`全ての条件を満たすバージョン`が利用可能となります。
ただし、それぞれの指定が矛盾している場合はエラーが発生します。

### 例➀.複数の箇所でterraformバージョンが指定されている場合
以下であれば、`1.4.0 以上、1.5.0 未満`が利用可能なTerraformのバージョンとなります。
```terraform:Root Module
terraform {
  required_version = ">= 1.3.0, < 1.5.0"
}
```

```terraform:Module A
terraform {
  required_version = ">= 1.4.0, < 1.6.0"
}
```

以下のような場合は共通する範囲が無く矛盾しており、
root moduleのバージョン指定に合わせていますが、terraform init の段階でmodule側でエラーが発生します。

```terraform:Root Module
terraform {
  required_version = ">= 1.3.0, < 1.5.0"
}
```

```terraform:Module A
terraform {
  required_version = ">= 1.6.0"
}
```

```text:エラーメッセージ
│ Error: Unsupported Terraform Core version
│ 
│   on ../modules/s3/main.tf line 2, in terraform:
│    2:   required_version = ">= 1.6.0"
│ 
│ Module module.s3 (from ../modules/s3) does not support Terraform version 1.4.0. To proceed, either choose another supported Terraform version or update this version constraint. Version constraints
│ are normally set for good reason, so updating the constraint may lead to other errors or unexpected behavior.
```

### 例➁.複数の箇所でproviderバージョンが指定されている場合
以下であれば、`4.15.0 以上 4.20.0 未満`が利用可能なaws providerのバージョンとなります。

```terraform:Root Module
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.10.0, < 4.20.0"
    }
  }
}
```

```terraform:Module A
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.15.0, < 4.25.0"
    }
  }
}
```

以下のような場合は共通する範囲が無く矛盾しているため、terraform init の段階でエラーが発生します。

```terraform:Root Module
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.10.0, < 4.15.0"
    }
  }
}
```

```terraform:Module A
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.16.0"
    }
  }
}
```

```text:エラーメッセージ
│ Error: Failed to query available provider packages
│ 
│ Could not retrieve the list of available versions for provider hashicorp/aws: no available releases match the given constraints >= 4.10.0, < 4.15.0, >= 4.16.0
```

### 例➂.複数の箇所でterraformバージョン、providerバージョンが指定されている場合
以下であれば`1.4.0 以上 1.5.0 未満`が利用可能なTerraformのバージョン、
`4.15.0 以上 4.20.0 未満`が利用可能なaws providerのバージョンとなります。

```text:Terraformバージョンの指定
Root Module: >= 1.3.0, < 1.5.0
Module A   : >= 1.4.0, < 1.6.0
```

```text:Providerバージョンの指定
Root Module: aws >= 4.10.0, < 4.20.0
Module A   : aws >= 4.15.0, < 4.25.0
```

両方の条件が満たされる場合、Terraformは両方の条件を満たすバージョンのみを選択します。
両方の条件が矛盾する場合 いずれかの制約が満たされない場合、エラーとなります。

# 結局どのようにバージョンを指定すればよいのか…？
ここまでの説明が少し長くなってしまいましたが、最終的に以下のポイントを押さえてバージョンを指定すれば大きな問題が起きることはないでしょう。

* root moduleでは必ずバージョンを指定して、プロジェクト全体の一貫性を保つ
* 他プロジェクトと共有するmodule(ローカル)であればバージョンを指定する
* module(リモート)が特定のバージョンを要求する場合、root moduleと矛盾しない範囲でバージョンを指定する

Terraformのバージョン管理は初めは複雑に感じるかもしれませんが、適切な設定を行うことでその後のチーム開発や運用が格段に楽になります。
この記事が、皆様のTerraform Lifeを少しでも快適にするお役に立てれば幸いです。
