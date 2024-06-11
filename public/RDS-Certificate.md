---
title: RDSの認証局(rds-ca-2019)、もう更新した？？
tags:
  - AWS
  - RDS
  - CLI
private: false
updated_at: '2023-10-12T23:53:04+09:00'
id: afc311811da8978751a4
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
<!-- 発端や概要を記載 -->
2023年8月末に、AWSよりAmazon RDS認証局証明書rds-ca-2019の有効期限に関する通知がありました。
通知内容は、
「RDS認証局の証明書rds-ca-2019は2024/8/22に証明書の有効期限が切れるので、暗号化通信を利用している場合は認証局を更新する必要がある。」
とのことで、詳細は以下の公式ドキュメントに記載されています。

[RDS認証局更新について-公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL-certificate-rotation.html)

実はこれ、過去に「rds-ca-2015」から「rds-ca-2019」へ更新が発生した際も同様の周知がされていました。
しかし私は完全に油断しており、Health Dashboardの通知が来てから思い出しました...。
基本的に「rds-ca-2015」のときとやることは一緒ですが、今一度確認項目と更新方法を振り返ろうと思います。

なお以下の通りリージョンによって有効期限が異なるため、必ずしも2024/8/23までに対応が必要になるわけではないです。
| rds-ca-2019 有効期限   | リージョン                  |
|:----------------------|:---------------------------|
| 2024 年 5 月 8 日     | 中東 (バーレーン)            |
| 2024 年 8 月 22 日    | 米国東部 (オハイオ)、米国東部 (バージニア北部)、米国西部 (北カリフォルニア)、米国西部 (オレゴン)、アジアパシフィック (ムンバイ)、アジアパシフィック (大阪)、アジアパシフィック (ソウル)、アジアパシフィック (シンガポール)、アジアパシフィック (シドニー)、アジアパシフィック (東京)、カナダ (中部)、欧州 (フランクフルト)、欧州 (アイルランド)、欧州 (ロンドン)、欧州 (ミラノ)、欧州 (パリ)、欧州 (ストックホルム)、および南米 (サンパウロ)     |
| 2024 年 9 月 9 日     | 中国 (北京)、中国 (寧夏)    |
| 2024 年 10 月 26 日   | アフリカ (ケープタウン)     |
| 2024 年 10 月 28 日   | 欧州 (ミラノ)              |
| 2061 年まで影響なし    | アジアパシフィック (香港)、アジアパシフィック (ハイデラバード)、アジアパシフィック (ジャカルタ)、アジアパシフィック (メルボルン)、欧州 (スペイン)、欧州 (チューリッヒ)、イスラエル (テルアビブ)、中東 (UAE)、AWS GovCloud (米国東部)、および AWS GovCloud (米国西部) |

詳細は以下公式ドキュメントをご確認ください。

* [リージョンごとの有効期限について](https://aws.amazon.com/jp/blogs/news/rotate-your-ssl-tls-certificates-now-amazon-rds-and-amazon-aurora-expire-in-2024/#:~:text=%E5%BD%B1%E9%9F%BF%E3%82%92%E5%8F%97%E3%81%91%E3%82%8B%E3%83%AA%E3%83%BC%E3%82%B8%E3%83%A7%E3%83%B3,%E3%82%92%E6%AC%A1%E3%81%AB%E7%A4%BA%E3%81%97%E3%81%BE%E3%81%99%E3%80%82)

# 目次
<!-- タイトルとアンカー名を編集 -->
1. [RDSの作成状況に応じた対応](#1-RDSの作成状況に応じた対応)
2. [暗号化通信の確認](#2-暗号化通信の確認)
3. [利用できる認証局](#3-利用できる認証局)
4. [認証局の更新方法](#4-認証局の更新方法)
5. [最後に](#5-最後に)

<!-- 各チャプター -->
<a id="#Chapter1"></a>
# 1-RDSの作成状況に応じた対応

まず今回のRDS認証局の更新について、
RDSの作成状況に応じて以下の2パターンに対応が別れます。
そして本記事では「2.RDS作成後」を対象としています。

```
(1)RDS作成前
　⇒作成時に「rds-ca-2019」以外の認証局を選択する
(2)RDS作成後
　⇒「rds-ca-2019」以外の認証局へ更新する
```

なおこれからRDSを作成する場合、
現状(23年10月)では、RDS作成画面にてデフォルトで「rds-ca-2019」が選択されています。
そのためGUIでRDSを作成される場合は要注意です...。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/38396109-a492-413c-7155-5764ab526f6b.png)

また選ぶべき認証局については、本記事中の[3.利用できる認証局](#3.利用できる認証局)が参考になるかと思います。

<a id="#Chapter2"></a>
# 2-暗号化通信の確認

公式ドキュメントでは以下の記述があり、
「RDSへの接続において、SSL/TLS通信を利用している場合」に認証局の更新が必要と確認できます。
```
RDS DB インスタンスへの接続に証明書検証付きの Secure Sockets Layer (SSL) 
または Transport Layer Security (TLS) を使用しているか、使用する予定がある場合は、
新しい CA 証明書 rds-ca-rsa2048-g1、rds-ca-rsa4096-g1、または rds-ca-ecc384-g1 のいずれかの使用を検討する必要があります。
```

そのためまずは以下の方法で、暗号化通信の利用状況を確認しましょう。

### ①DBに接続して確認
DBに接続後、暗号化通信を利用しているか確認します。
以下は確認コマンドと、暗号化通信を利用している場合の出力例です。

#### MySQL、MariaDB
```sql
> SHOW STATUS LIKE 'Ssl_cipher';

+---------------+--------------------+
| Variable_name | Value              |
+---------------+--------------------+
| Ssl_cipher    | DHE-RSA-AES256-SHA |
+---------------+--------------------+
```
#### PostgreSQL
```sql
>SELECT version(), 
       ssl, 
       cipher, 
       key_bits, 
       clientdn 
FROM pg_stat_ssl 
WHERE pid = pg_backend_pid();

 version | ssl |      cipher       | key_bits | clientdn 
---------+-----+-------------------+----------+----------
 13.1    | t   | TLS_AES_256_GCM_SHA384 |    256   | (null)
```
#### Oracle
```sql
>SELECT NETWORK_SERVICE_BANNER 
FROM v$session_connect_info 
WHERE SID = (SELECT SID FROM v$mystat WHERE ROWNUM = 1);

NETWORK_SERVICE_BANNER
-------------------------------------------------
TCP/IP NT Protocol Adapter for Linux: Version 19.0.0.0.0 - Production
SSL/TLS:  (TLSv1.2,128-bit AES256-SHA256)
```
#### SQL Server
```sql
>SELECT encrypt_option 
FROM sys.dm_exec_connections 
WHERE session_id = @@SPID;

 encrypt_option
---------------
 TRUE
```

### ②パラメータグループ/オプショングループで確認
DBエンジンにより異なりますが、パラメータグループやオプショングループで暗号化通信に関する設定があります。

#### MySQL、MariaDB (パラメータグループ)
| 設定項目                | 設定内容                   |
|:------------------------|:---------------------------|
|require_secure_transport	|すべての接続にSSL/TLSを要求  |

#### PostgreSQL (パラメータグループ)
| 設定項目                | 設定内容                   |
|:------------------------|:---------------------------|
|rds.force_ssl	|すべての接続にSSL/TLSを要求            |

#### Oracle (オプショングループ)
| 設定項目           | 設定内容                      |
|:-------------------|:-----------------------------|
|SQLNET.SSL_VERSION	 |利用するSSL/TLSバージョンの指定 |
|SQLNET.CIPHER_SUITE |利用する暗号化スイートの指定    |

#### SQL Server (パラメータグループ)
| 設定項目                | 設定内容                   |
|:------------------------|:---------------------------|
|rds.force_ssl	|すべての接続にSSL/TLSを要求            |


上記の確認の結果、`暗号化通信を行っている` or `暗号化通信を必須とする設定がされている`場合、RDSの認証局の更新作業が必要となります。

なお暗号化通信を行っていなければ認証局の更新は不要と思われますが、
更新しない場合、いつまでもマネジメントコンソール画面の「証明書の更新」にリストアップされて煩わしいので、
個人的には暗号化通信を利用していない場合でも、更新したほうがbetterだと思います...。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/1840ce73-123d-a7ca-4427-948b83814384.png)

<a id="#Chapter3"></a>
# 3-利用できる認証局

以下が利用できる認証局の一覧です。
この中からさらに、DBエンジンごとに利用できる認証局が決まっています。

#### 認証局一覧
| 証明書          | 秘密鍵アルゴリズム | 署名アルゴリズム  | 証明機関の期限 |
|:----------------|:------------------|:-----------------|:--------------|
|rds-ca-2019	    |RSA2048            |	SHA256           |	2024/8/23    |
|rds-ca-rsa2048-g1|	RSA2048           |	SHA256           |	2061/5/26    |
|rds-ca-rsa4096-g1|	RSA4096           |	SHA384           |	2121/5/26    |
|rds-ca-ecc384-g1 |	ECC384            |	SHA384           |	2121/5/26    |

#### DBエンジンごとに利用できる認証局
| DBエンジン     | 利用できる認証局                                                            |
|:---------------|:---------------------------------------------------------------------------|
|MySQL、MariaDB	 |`rds-ca-2019`, `rds-ca-ecc384-g1`, `rds-ca-rsa4096-g1`, `rds-ca-rsa2048-g1` |
|PostgreSQL	     |`rds-ca-2019`, `rds-ca-ecc384-g1`, `rds-ca-rsa4096-g1`, `rds-ca-rsa2048-g1` |
|Oracle	         |`rds-ca-2019`, `rds-ca-rsa4096-g1`, `rds-ca-rsa2048-g1`                     |
|SQL Server	     |`rds-ca-2019`, `rds-ca-rsa4096-g1`, `rds-ca-rsa2048-g1`                     |

* [認証局の一覧-公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html)

なおクライアントソフトやアプリケーションによっては、「rds-ca-rsa4096-g1」、「rds-ca-ecc384-g1」などの認証局の暗号化方式に対応していない可能性があるため、
基本的にはAWSが推奨している「rds-ca-rsa2048-g1」を選択すれば問題ないかと思います。

#### rds-ca-2019の記述抜粋
```
この CA は 2024 年に有効期限が切れ、サーバー証明書の自動ローテーションはサポートされていません。
この CA を使用して同じ標準を維持したい場合は、rds-ca-rsa2048-g1 CA に切り替えることをお勧めします。
```



<a id="#Chapter4"></a>
# 4-認証局の更新方法

更新作業について、以下の2つの作業に大きく分かれます。
```
(1)アプリケーションのSSL/TLS 証明書の更新
(2)RDSインスタンスの認証局の更新
```

#### ①アプリケーションのSSL/TLS 証明書の更新
公式ドキュメントの古い証明書バンドルをクライアントソフトやアプリケーションで利用している場合、
証明書バンドルの更新が必要になる可能性があります。
また具体的な設定方法はアプリケーションに依存するため、本記事では割愛します。

なお証明書バンドルを利用する設定をしていなければ、基本的にこちらの対応は不要となります。

* [証明書バンドル ダウンロード/設定方法 - 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html)
#### ②RDSインスタンスの認証局の更新
まず認証局更新時に、DBエンジンやバージョンによって再起動する場合と、しない場合があります。
再起動の有無はマネジメントコンソールやAWS CLIで確認ができ、以下はAWS CLIでMySQLを確認した例です。
出力結果の`SupportsCertificateRotation`が`true`であれば、再起動が発生せず、
`SupportsCertificateRotation`が`false`の場合は自動的に再起動が発生します。
```shell
aws rds describe-db-engine-versions \
--engine mysql \
--query "DBEngineVersions[].{Engine:Engine, EngineVersion:EngineVersion,SupportsCertificateRotation:SupportsCertificateRotationWithoutRestart}"
```
出力例：
```json
[
  {
    "Engine": "mysql",
    "EngineVersion": "8.0.34",
    "SupportsCertificateRotation": true
  },
  {
    "Engine": "mysql",
    "EngineVersion": "5.7.43",
    "SupportsCertificateRotation": false
  }
]
```

次に認証局の更新方法ですが、マネジメントコンソールから更新する方法とAWS CLIで更新する方法があります。
* [RDS認証局更新について-公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL-certificate-rotation.html)

今回はAWS CLIで更新します。
またコマンドのオプションにて、
`即時更新`、`次回のメンテナンスウィンドウで更新`のどちらかを指定できます。

##### (1)即時更新
```shell
aws rds modify-db-instance \
    --db-instance-identifier $RdsInstanceName \
    --ca-certificate-identifier rds-ca-rsa2048-g1 \
    --no-apply-immediately
```

##### (2)次回のメンテナンスウィンドウにて更新
```shell
aws rds modify-db-instance \
    --db-instance-identifier $RdsInstanceName \
    --ca-certificate-identifier rds-ca-rsa2048-g1 \
    --apply-immediately
```

<a id="#Chapter5"></a>
# 5-最後に
以上で認証局の更新作業は終わりです。
振り返ってみると更新作業自体はすぐに完了しますが、
暗号化通信の利用状況の確認に時間がかかりそうですね...。

なお次回の更新は最短でも`rds-ca-rsa2048-g1`の`2061/5/26`であり、
私はまた十中八九忘れているので、そのときはこの記事を見て思い出すのでしょう...。
