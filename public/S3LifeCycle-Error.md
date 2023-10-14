---
title: 【小ネタ】S3のライフサイクルルールで、非現行バージョンの保持するバージョン数の設定ができない...？？
tags:
  - AWS
  - S3
private: false
updated_at: '2023-08-04T00:32:03+09:00'
id: 2cecece1a8aabf235932
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
先日バージョニングを有効化しているS3で以下のライフサイクルルールを設定したところ...
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/0807183f-b2da-6cf0-535d-dc80338faa26.png)


ルールの作成時にエラーが出ました。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/d2570365-0e4a-9735-4836-284292936396.png)



直訳すると以下の通りですが、
NewerNoncurrentVersionsとは一体なんだろう...。
```:翻訳
'NewerNoncurrentVersions' for
NoncurrentVersionExpiration action must be a positive integer
↓
NoncurrentVersionExpiration アクションの
「NewerNoncurrentVersions」は正の整数である必要があります
```

# 結論
「NewerNoncurrentVersions」＝「保持する新しいバージョンの数」です。
なので非現行バージョンのオブジェクトは1バージョンも保持しない、という場合は保持する新しいバージョンの数を空欄で作成しましょう。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/be0b8079-8d46-070b-e026-e3f2086ab3f5.png)

ちなみに公式ドキュメントでは以下の記載があり、
AWSとしては保持する新しいバージョン数はオプション設定なので、
そもそも値を入力する必要がないということなのでしょう...。
> オプションで、[Number of newer versions to retain] (保持する新しいバージョンの数) に値を入力して、保持する新しいバージョンの数を指定できます。
- [バケットのライフサイクル設定](https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/how-to-set-lifecycle-configuration-intro.html)
