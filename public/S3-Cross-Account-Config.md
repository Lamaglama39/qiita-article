---
title: S3クロスアカウント連携の設定パターン
tags:
  - AWS
  - S3
  - CLI
private: false
updated_at: '2023-10-27T01:06:45+09:00'
id: 6321606c39200a28bd3c
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに
<!-- 発端や概要を記載 -->
本記事はS3のクロスアカウント設定をする度に調べている忘れっぽい私のための、個人用にまとめた記事です。
動作確認だけしたい方は以下にTerraformを用意しているので、ご利用ください。

- [本記事のTerraformコード](https://github.com/Lamaglama39/terraform-for-aws/tree/main/s3-multi-account-deploy)

# 目次
<!-- タイトルとアンカー名を編集 -->
1. [設定パターンについて](#1-設定パターンについて)
2. [AssumeRoleを利用しない場合](#2-assumeroleを利用しない場合)
3. [AssumeRoleを利用する場合](#3-assumeroleを利用する場合)
4. [参考文献](#6-参考文献)

<a id="#Chapter1"></a>
# 1-設定パターンについて

アカウント間でS3連携の設定をする場合、主に以下の設定パターンがあります。
本記事では①、②の方法が対象です。

---
### ①AssumeRoleを利用しない場合
![don't-use-assumerole-image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/298e48e8-2432-345a-ddc3-5a125e487d50.png)

---
### ②AssumeRoleを利用する場合
![use-assumerole-image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/9e78f396-1b8e-f103-8924-1c2acc48c8a7.png)

---
### ③レプリケーション
[オブジェクトのレプリケーション-公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/replication.html)

---

なお「そもそもAssumeRoleとはなんだい？」という方は以下の記事をご参照ください、めちゃくちゃわかりやすいのでおすすめです...。

[IAM ロールの PassRole と AssumeRole をもう二度と忘れないために絵を描いてみた](https://dev.classmethod.jp/articles/iam-role-passrole-assumerole/)

<a id="#Chapter2"></a>
# 2-AssumeRoleを利用しない場合

#### (1) アカウント1 EC2用IAMロール設定
アカウント①EC2にアタッチするIAMロールに、アカウント①S3バケット、アカウント②S3バケットへアクセスする権限を設定します。
ResourceにS3バケットのARNを適用します。

```json:IAMポリシー
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Account1S3Access",
            "Action": [
                "s3:PutObject",
                "s3:ListBucket",
                "s3:GetObject"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::<アカウント1-S3バケット名>/*",
                "arn:aws:s3:::<アカウント1-S3バケット名>"
            ]
        },
        {
            "Sid": "Account2S3Access",
            "Action": [
                "s3:PutObject",
                "s3:ListBucket",
                "s3:GetObject"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::<アカウント2-S3バケット名>/*",
                "arn:aws:s3:::<アカウント2-S3バケット名>"
            ]
        }
    ]
}
```
#### (2) アカウント2 S3バケットポリシー設定
アカウント②S3バケットポリシーに、アカウント①EC2のIAMロールからアクセスする権限を設定します。
Principalにアカウント①EC2で利用しているIAMロールのARN、
Resourceにアカウント②S3バケットのARNを適用します。
```json:バケットポリシー
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Allow Account1 IamRole",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<アカウント1-ID>:role/<アカウント1-IAMロール名>"
            },
            "Action": [
                "s3:PutObject",
                "s3:ListBucket",
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::<アカウント2-S3バケット名>/*",
                "arn:aws:s3:::<アカウント2-S3バケット名>"
            ]
        }
    ]
}
```

<a id="#Chapter3"></a>
# 3-AssumeRoleを利用する場合

#### (1) アカウント1 EC2用IAMロール設定
アカウント①EC2にアタッチするIAMロールに、アカウント②IAMロールへAssumeRoleする権限を設定します。
Resourceにアカウント②IAMロールのARNを適用します。

```json:IAMポリシー
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Account1AssumeRole",
            "Action": "sts:AssumeRole",
            "Effect": "Allow",
            "Resource": "arn:aws:iam::<アカウント2-ID>:role/<アカウント2-IAMロール名>",
        }
    ]
}
```

#### (2) アカウント2 AssumeRole用IAMロール設定
アカウント②IAMロールに、アカウント①EC2のIAMロールからAssumeRoleする権限を設定します。
IAMポリシーのResourceにアカウント②S3バケットのARN、
IAMロールの信頼ポリシーのPrincipalnにアカウント①のIAMロールのARNを適用します。

```json:IAMポリシー
{
    "Sid": "Account2S3Access",
    "Statement": [
        {
            "Action": [
                "s3:PutObject",
                "s3:ListBucket",
                "s3:GetObject"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::<アカウント2-S3バケット名>/*",
                "arn:aws:s3:::<アカウント2-S3バケット名>"
            ]
        }
    ],
    "Version": "2012-10-17"
}
```
```json:IAMロール 信頼ポリシー
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<アカウント1-ID>:role/<アカウント1-IAMロール名>"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

#### (3) アカウント1 S3バケットポリシー設定
アカウント①S3バケットポリシーに、アカウント②IAMロールからアクセスする権限を設定します。
Principalにアカウント②IAMロールのARN、
Resourceにアカウント①S3バケットのARNを適用します。
```json:バケットポリシー
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Allow Account2 IamRole",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<アカウント2-ID>:role/<アカウント2-IAMロール名>"
            },
            "Action": [
                "s3:PutObject",
                "s3:ListBucket",
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::<アカウント1-S3バケット名>/*",
                "arn:aws:s3:::<アカウント1-S3バケット名>"
            ]
        }
    ]
}
```


#### (4) EC2 Config設定
アカウント①EC2へログインして、$HOME/.aws/config へ以下の設定を追加します。
```text:.aws/config
[profile account2]
role_arn = arn:aws:iam::<アカウント2-ID>:role/<アカウント2-IAMロール名>
credential_source = Ec2InstanceMetadata
```
設定後、以下のようにAWS CLI使用時にProfileを指定することで実行できます。
```bash:コマンド
aws s3 cp \
  s3://<アカウント1-S3バケット名> \
  s3://<アカウント2-S3バケット名> \
  --profile account2
```

<a id="#reference"></a>
## 6-参考文献
- [バケットポリシー-公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/access-policy-language-overview.html)
- [IAMポリシー-公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/user-policies.html)
