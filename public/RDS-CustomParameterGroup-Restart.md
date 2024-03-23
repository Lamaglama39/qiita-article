---
title: 【RDS】カスタムパラメータグループと再起動に関する小話
tags:
  - AWS
  - RDS
  - Aurora
private: false
updated_at: '2024-03-13T22:36:37+09:00'
id: 4e3a803bfbcb3e980ab4
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- 発端や概要を記載 -->
# はじめに
RDSの利用において欠かせないリソースのひとつがパラメータグループです。
何故ならRDSではDBエンジン問わずパラメータグループを通して各種設定を行い、カスタマイズを行う上で必須の要素だからです。

しかしパラメータグループを利用する上でDBインスタンスの再起動は避けては通れません。
特にリリース後のシステムでRDSの設定変更を行う場合、再起動が必要かどうかは大きな問題になりえます...。

そこで本記事では多くの方が意識していないかもしれない、
カスタムパラメータグループを使用する際に再起動が発生するタイミングを解説します。

<!-- 各チャプター -->
<a id="#Chapter1"></a>

# 1.パラメータグループとは
そもそもパラメータグループとはなんでしょうか？
**今更説明されなくても知ってるよ...** という方は[3.再起動が発生するタイミング](#3再起動が発生するタイミング)に飛んでください。

まずAWS公式ドキュメントには以下の一文があります。
```
DB パラメータグループは、
1 つ以上の DB インスタンスに適用されるエンジン設定値のコンテナとして機能します。
```
* [パラメータグループ 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/parameter-groups-overview.html)

これはどういう事かというと、
`データベースの設定値をパラメータグループという形で管理している`ということです。
これによりデータベースの設定ファイルを編集することなく、
`マネジメントコンソール` or `AWS CLI` or `AWS SDK` などを使用して簡単に設定することを実現しています。

言うまでもないですがこれは大きな利点であり、
例えば従来の自己管理型のデータベース（例えばEC2インスタンス上にインストールしたMySQL）であれば、
Linuxの場合はmy.cnf、Windowsの場合はmy.iniといった設定ファイルを直接編集する必要があります。
こうした作業では以下のような手順が必要となり、大変手間がかかります...。
```
1.設定ファイルをコピーしてバックアップを作成
2.設定ファイルを編集
3.バックアップした設定ファイルと編集した設定ファイルをdiffで比較
4.DBエンジンを再起動
```

結論としてRDSのパラメータグループを使用する主な利点は以下の三点です、
これにより運用の効率化や設定ミスのリスク軽減につながります。

* 設定変更が簡単であること
* 複数のデータベースインスタンス間で設定を容易に共有できること
* 設定変更が即時に適用されるか、再起動後に適用されるかをAWSが明示してくれること


<a id="#Chapter2"></a>

# 2.パラメータグループの種類について
パラメータグループには以下の二種類があり、設定対象、および設定項目が異なります。

#### 設定対象/対応DBエンジン
| パラメータグループ種別        | 設定対象                              | 対応DBエンジン                        |
|:----------------------------|:-----------------------------------------|:--------------------------------------|
|	DBパラメータグループ          | RDSインスタンス                           | MySQL、PostgreSQL、Oracle、SQL Server、Db2 |
|	DBクラスターパラメータグループ | マルチ AZ DB クラスター、Auroraクラスター   | MySQL、PostgreSQL     |

#### 設定項目の例
| パラメータグループ種別        | 設定項目の例         |
|:----------------------------|:-------------------|
|	DBパラメータグループ          | `同時接続の最大数`、`InnoDBバッファプールのサイズ`など、インスタンスのパフォーマンスや動作に直接関連するパラメータの設定 |
|	DBクラスターパラメータグループ | `サーバのデフォルト文字セット`、`サーバのデフォルト照合順序`など、クラスタ全体で一貫した動作を要求するパラメータの設定   |

* [DB パラメータグループ 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/USER_WorkingWithDBInstanceParamGroups.html)
* [DB クラスターパラメータグループ 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/USER_WorkingWithDBClusterParamGroups.html)

このように、DBパラメータグループとDBクラスターパラメータグループは、
設定対象と適用されるRDSのデータベースのタイプによって区別されます。

<a id="#Chapter3"></a>

# 3.再起動が発生するタイミング
前置きが長くなりましたが、
カスタムパラメータグループを利用する際のRDSの再起動について大きく以下の3つのパターンがあります。 
```
1. デフォルトパラメータグループからカスタムパラメータグループへ変更するとき
2. カスタムパラメータグループの設定値を変更するとき
3. 停止状態のRDSを起動するとき
```

## ①デフォルトパラメータグループからカスタムパラメータグループへ変更するとき
まずデフォルトパラメータグループからカスタムパラメータグループに切り替える行為によって、自動的な再起動は発生しません。
ただしカスタムパラメータグループの設定を適用するには再起動する必要があり、
これはメンテナンスウィンドウで反映することができないため、`手動でDBインスタンスを再起動する`必要があります。

実際にデフォルトパラメータグループからカスタムパラメータグループへ変更すると、
RDSのマネジメントコンソールでは以下のステータスが表示されます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/a4784fd5-b37b-fb9a-4c99-cdac08c9399f.png)

上記の状態から手動で再起動を行うことで、以下のステータスへ移行しカスタムパラメータグループを適用している状態となります。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/a501bc4f-efba-d16c-bd6e-adc2b34defc1.png)


## ②カスタムパラメータグループの設定値を変更するとき
カスタムパラメータグループの設定値を変更する際に、設定項目によっては適用するために再起動が必要となります。
具体的には`動的パラメータ`と`静的パラメータ`があり、それぞれ再起動の要否が異なります。

* 動的パラメータ - インスタンスの再起動なしに即時適用されます。
* 静的パラメータ - 変更を有効にするためにインスタンスの再起動が必要になります。

なお各パラメータが動的か静的はマネジメントコンソールにて`タイプの適用`で確認できますが...、
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/04b874d4-80b5-4aab-c004-24449c16e2ed.png)

以下のAWS CLIで一括出力できるので、まとめて確認したい場合はこちらの方法をおすすめします。
```bash:
$ aws rds describe-db-parameters \
    --db-parameter-group-name $PARAMETER_GROUP_NAME
```

また以下のように`--query`オプションで`ApplyType`が`static`を指定することで静的パラメータのみ出力できます。
またついでにjqを利用してCSV形式でファイルに出力しています。
```bash:
$ aws rds describe-db-parameters \
    --db-parameter-group-name $PARAMETER_GROUP_NAME \
    --query "Parameters[?ApplyType=='static']" \
    --output json | 
    jq -r '
    ["ParameterName","ParameterValue","Description","Source","ApplyType","DataType","AllowedValues","IsModifiable","ApplyMethod"],
    ( .[] | [.ParameterName, .ParameterValue, .Description, .Source, .ApplyType, .DataType, .AllowedValues, .IsModifiable, .ApplyMethod]) | @csv' > parameter.csv
```

出力したファイルはエクセルで以下のような感じで見れるので、
良ければご利用ください。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/6884b1a5-871e-14a1-a72d-e182d1b63b9e.png)


## ③停止状態のRDSを起動するとき
実はこれが本記事で一番伝えたかった再起動です、
とりあえず論より証拠ということで以下のログをご覧ください。

まず以下は`デフォルトパラメータグループ`を利用していて、停止しているRDSを起動した際のイベントログです。
```text:デフォルトパラメータグループを利用
March 13, 2024, 00:40 (UTC+09:00)	Recovery of the DB instance has started. Recovery time will vary with the amount of data to be recovered.
March 13, 2024, 00:40 (UTC+09:00)	Recovery of the DB instance has started. Recovery time will vary with the amount of data to be recovered.
March 13, 2024, 00:43 (UTC+09:00)	DB instance restarted
March 13, 2024, 00:43 (UTC+09:00)	DB instance restarted
March 13, 2024, 00:43 (UTC+09:00)	Recovery of the DB instance is complete.
March 13, 2024, 00:45 (UTC+09:00)	DB instance started
```

次に以下は`カスタムパラメータグループ`を利用していて、停止しているRDSを起動した際のイベントログです。
```text:カスタムパラメータグループを利用
March 13, 2024, 00:39 (UTC+09:00)	Recovery of the DB instance has started. Recovery time will vary with the amount of data to be recovered.
March 13, 2024, 00:40 (UTC+09:00)	Recovery of the DB instance has started. Recovery time will vary with the amount of data to be recovered.
March 13, 2024, 00:40 (UTC+09:00)	Recovery of the DB instance has started. Recovery time will vary with the amount of data to be recovered.
March 13, 2024, 00:43 (UTC+09:00)	DB instance restarted
March 13, 2024, 00:43 (UTC+09:00)	DB instance restarted
March 13, 2024, 00:43 (UTC+09:00)	Recovery of the DB instance is complete.
March 13, 2024, 00:44 (UTC+09:00)	DB instance started
March 13, 2024, 00:45 (UTC+09:00)	DB instance shutdown
March 13, 2024, 00:45 (UTC+09:00)	DB instance restarted
March 13, 2024, 00:45 (UTC+09:00)	DB instance restarted
```

ご覧の通り、カスタムパラメータグループを利用したRDSの場合、
`DB instance started`でRDSが起動した後に`DB instance shutdown`、および`DB instance restarted`が発生しており、
RDSの再起動が発生していることが分かります。
しかもこれは`カスタムパラメータグループを利用している場合は必ず発生する事象`です。

ですが実は、この挙動は以下の通り公式ドキュメントにがっつり書いております。
また具体的に再起動する理由が書いてあるわけではないのであくまで推測となりますが、
カスタムパラメータグループを適用する過程で再起動を行っていると考えられます。
```
注記
カスタムパラメータグループを使用するように DB インスタンスを変更して、
DB インスタンスを起動すると、RDS は起動プロセスの一環として DB インスタンスを自動的に再起動します。
```
* [パラメータグループ 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AmazonRDS/latest/UserGuide/parameter-groups-overview.html)

そしてこれの一体何が問題かというと、この仕様を知らずにRDSの監視設定をしている場合です。
例えばRDSの起動時間の直後(10分前後など)から死活監視をしていると十中八九引っかかります...。

~~もちろん私はまんまとハマっています。~~

<a id="#Chapter4"></a>

# 4.最後に
以上、カスタムパラメータグループと再起動に関する小話でした。  

一部常識的なものもありましたが、
皆様とRDSとの生活が少しでもより良いものになれば幸いです。


<a id="#Chapter5"></a>

## 参考ドキュメント
本記事の作成に際して、以下のドキュメントおよび記事を参考にさせていただきました。
この場を借りてお礼申し上げます。

* [Amazon RDS DB パラメータグループの値の変更方法を教えてください。](https://repost.aws/ja/knowledge-center/rds-modify-parameter-group-values)
* [RDSのデフォルトパラメータグループを変更する](https://blog.denet.co.jp/rds-parametergroup/)
* [RDS の動的パラメータ変更時に再起動は必要なのか教えてください](https://dev.classmethod.jp/articles/tsnote-rds-change-dynamic-parameter-reboot-01/)
