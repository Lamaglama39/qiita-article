---
title: アカウント内のS3バケット容量を一括出力する方法
tags:
  - AWS
  - S3
  - aws-cli
private: false
updated_at: '2023-07-16T09:07:47+09:00'
id: cc4a6488eaeba30cdbbe
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
AWSでしばらく遊んでいると、いつの間にか謎のS3バケットが増えていきませんか？
私はめんどくさがりで消すのをよく忘れるので、使途不明なS3が日々発生しています。
そんな中重い腰を上げて、一つ一つS3バケットの容量を見て要否を確認するのはさらに面倒...。

というわけで、AWS CLIでオブジェクト数と容量をまとめて出力する方法を共有します。

<!-- 各チャプター -->
<a id="#コマンド"></a>
# コマンド
以下のコマンドでまとめて出力できます。
大事なのはaws s3 lsコマンドの以下オプションです。
- --recursive
指定したS3バケット配下のすべてのオブジェクトに対して再帰的に実行してくれる。
- --human-readable
KiBやMiBなど、確認しやすいサイズ表記にしてくれる。
- --summarize
`Total Objects`と`Total Size`を表示してくれる。

```shell:実行
for bucket in $(aws s3 ls | awk '{print $3}'); do
  echo $bucket
  aws s3 ls s3://$bucket --recursive --human-readable --summarize | tail -n 2
  echo -e
done
```
```terminal:出力例
bucket-1
Total Objects: 41
   Total Size: 1.4 MiB

bucket-2
Total Objects: 402
   Total Size: 123.4 MiB

bucket-3
Total Objects: 19
   Total Size: 145.2 KiB
```

<a id="#注意事項"></a>
# 注意事項
実はListリクエストは少額ですが料金がかかるため、
各S3に大量のオブジェクトが格納されている場合、費用面で注意が必要です。
具体的には、東京リージョンのスタンダードクラスの S3 バケットでは、`1,000 LIST リクエスト`あたり `0.0047 USD` の料金が発生します。

例として、Listリクエストは最大1,000オブジェクトまで取得できるので、
以下のようなイメージになります。
```
2,000,000(オブジェクト) / 1000(Listリクエスト上限) = 2
2 × 0.0047(USD) = 0.0094(USD)
```

<a id="#reference"></a>
# 参考文献
- [AWS CLI リファレンス](https://docs.aws.amazon.com/cli/latest/reference/s3/ls.html)
- [AWS CLI S3リクエスト料金](https://aws.amazon.com/jp/s3/pricing/)
