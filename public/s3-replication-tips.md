---
title: S3サーバーサイド暗号化によるレプリケーション時の権限設定の違い
tags:
  - AWS
  - S3
private: false
updated_at: '2025-01-20T12:35:07+09:00'
id: b20072eb78549092f3ec
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- 発端や概要を記載 -->
# はじめに
S3サーバーサイド暗号化はいくつか種類がありますが、それぞれレプリケーション時の権限設定が若干異なります。
なので設計の際に安易に暗号化方式を決めてしまうと、あとからレプリケーションを設定する際に困ることもあるでしょう。
~~(筆者は若干困りました…)~~

なので今回は`S3サーバーサイド暗号化ごとのレプリケーション時の権限設定の違い`を一覧化して記事にしました。

:::note info
本記事の内容は以下リポジトリで気軽に試せるので、よければご利用ください。
* [s3-bidirectional-replication](https://github.com/Lamaglama39/terraform-for-aws/tree/main/app/s3-bidirectional-replication)
:::


<!-- 各チャプター -->
<a id="#Chapter1"></a>

# S3サーバーサイド暗号化の種類
S3サーバーサイド暗号化は、`S3上に保存されるオブジェクトを自動で暗号化する機能`です。
これにより保存データのセキュリティレベル向上や、コンプライアンス要件への準拠が容易になります。
2024年12月時点で提供されているサーバーサイド暗号化は以下の通りです。

* [サーバー側の暗号化によるデータの保護](https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/serv-side-encryption.html)

| 名称                                                    | 特徴                                       | 管理負荷                  | コスト           |
|:--------------------------------------------------------|:-----------------------------------------|:---------------------------|:-----------------|
|	Amazon S3 マネージドキーを用いたサーバー側の暗号化 (SSE-S3) | AWS管理の暗号鍵を用いた暗号化方式 | キー管理を行う必要なし | 暗号化による管理コストは発生しない |
|	AWS KMSキーによるサーバー側の暗号化 (SSE-KMS)              | KMSキーに対して細かいアクセス制御が可能 | KMSキーのポリシー設定やアクセス権限管理が必要 | KMS本体、及びリクエストに応じてコスト発生 |
|	AWS KMSキーによる二層式サーバー側の暗号化 (DSSE-KMS)        | 二層の暗号化により高度なセキュリティ保護を提供 | SSE-KMSと同様 | SSE-KMSと同様 |
|	顧客提供のキーを用いたサーバー側の暗号化 (SSE-C)  | 利用者が暗号鍵を提供し、S3が暗号化/復号を行う | 利用者が鍵管理する必要あり | 暗号鍵の管理を行うため運用コストが発生 |


<a id="#Chapter2"></a>

# 暗号化ごとのレプリケーション設定
S3自体がレプリケーションを行っているため、`レプリケーション用のIAMロールが必要`となります。
また設定している暗号化方式に応じて、追加の設定が必要となります。

今回は例として、以下のような双方向レプリケーションを設定した状態を想定します。
同一のアカウント内で、別リージョンのS3にレプリケーションしています。

![s3-bidirectional-replication.drawio.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/1563f897-39e4-fcfe-2d67-c1d812f4f9ac.png)

### SSE-S3

SSE-S3の場合、`双方のS3バケットへのアクセス許可のみでレプリケーション可能`です。
デフォルトでS3がキー管理するため、レプリケーション先のバケットでも同様にSSE-S3として複製されます。

* [ライブレプリケーションのアクセス許可の設定](https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/setting-repl-config-perm-overview.html)

信頼ポリシーでは、
S3がIAMロールを引き受けるため、`s3.amazonaws.com`を指定します。

```json:IAM 信頼ポリシー
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Principal":{
            "Service":"s3.amazonaws.com"
         },
         "Action":"sts:AssumeRole"
      }
   ]
}
```

アクセス許可ポリシーでは、
`レプリケーション元/レプリケーション先のそれぞれのS3バケット`に対して許可設定します。

```json:IAM アクセス許可ポリシー
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect":"Allow",
         "Action":[
            "s3:GetReplicationConfiguration",
            "s3:ListBucket"
         ],
         "Resource":[
            "arn:aws:s3:::s3-bucket-tokyo"
         ]
      },
      {
         "Effect":"Allow",
         "Action":[
            "s3:GetObjectVersionForReplication",
            "s3:GetObjectVersionAcl",
            "s3:GetObjectVersionTagging"
         ],
         "Resource":[
            "arn:aws:s3:::s3-bucket-tokyo/*"
         ]
      },
      {
         "Effect":"Allow",
         "Action":[
            "s3:ReplicateObject",
            "s3:ReplicateDelete",
            "s3:ReplicateTags"
         ],
         "Resource":"arn:aws:s3:::s3-bucket-osaka/*"
      }
   ]
}
```

### SSE-KMS

SSE-KMSの場合、それぞれのS3バケットに対するアクセス許可の他に、
`暗号化に利用しているKMS鍵に対するアクセス許可`も必要となります。

* [暗号化されたオブジェクトのレプリケート (SSE-S3、SSE-KMS、DSSE-KMS、SSE-C)](https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/replication-config-for-kms-objects.html)


KMS キーポリシーはデフォルトの設定で問題ありません。

```json:KMS キーポリシー
{
    "Version": "2012-10-17",
    "Id": "default",
    "Statement": [
        {
            "Sid": "Enable IAM User Permissions",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::account-id:root"
            },
            "Action": "kms:*",
            "Resource": "*"
        }
    ]
}
```

IAM アクセス許可ポリシーではS3バケットに対するアクセス許可に加え、
以下のKMSキーに対するアクセス許可が必要となります。

* レプリケーション元のS3バケットで利用しているKMSキーに対しての`Decrypt`
* レプリケーション先のS3バケットで利用しているKMSキーに対しての`Encrypt`、`GenerateDataKey`

```json:IAM アクセス許可ポリシー
{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Effect": "Allow",
         "Action": ["kms:Decrypt"],
         "Resource": ["arn:aws:kms:ap-northeast-1:account-id:key/XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"]
      },
      {
         "Effect": "Allow",
         "Action": [
          "kms:Encrypt",
          "kms:GenerateDataKey"
          ],
         "Resource": ["arn:aws:kms:ap-northeast-3:account-id:key/XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"]
      }
   ]
}
```

### DSSE-KMS

SSE-KMSと同様の権限設定のため省略。

### SSE-C

SSE-S3と同様に、`双方のS3バケットへのアクセス許可のみでレプリケーション可能`です。
ただし`アップロード/ダウンロード時に利用者側で暗号鍵の指定が必要`となります。
以下はAWS CLIで実行した場合の例です。

```text:テストファイル/一時的な暗号鍵作成
echo "test text" > test.txt
SSE_CUSTOMER_KEY=$(openssl rand -hex 16)
```

```text:ファイルアップロード
aws s3 cp test.txt s3:// \
    --sse-c AES256 \
    --sse-c-key $SSE_CUSTOMER_KEY
```

```text:ファイルダウンロード
aws s3 cp s3://s3-bucket-tokyo/test.txt . \
    --sse-c AES256 \
    --sse-c-key $SSE_CUSTOMER_KEY
```


# まとめと所感
* SSE-S3: S3バケットに対するアクセス許可のみでレプリケーション可能
* SSE-KMS / DSSE-KMS: S3バケットに対するアクセス許可に加えて、KMSキーへのアクセス権限が必要
* SSE-C: S3バケットに対するアクセス許可のみでレプリケーション可能、ただしアップロード/ダウンロード時に暗号鍵の指定が必須

これらの特性を踏まえ、運用上の要件やセキュリティポリシー、コスト、運用負荷などを総合的に考慮して暗号化方式を選択してレプリケーション設定を行うことが重要です。
個人的には、`特に要件がなければデフォルトのSSE-S3`でいいかなと思います。
