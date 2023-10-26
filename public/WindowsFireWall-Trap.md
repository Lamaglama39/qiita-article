---
title: EC2でWindowsFireWallを利用する際の罠
tags:
  - AWS
  - EC2
  - SSM
  - Windows
private: false
updated_at: ""
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

<!-- 発端や概要を記載 -->

皆様はEC2で WindowsFireWall を利用していますか？
通常であればEC2のアクセス制御はOS問わず、
セキュリティグループやネットワーク ACL、または WAF など AWS 上のリソースで行うため、
よほど入り組んだ要件がなければ使用しないかと思います。

そんな WindowsFireWall について以前やらかしたことを思い出したので共有します...。
なお本記事の対象はEC2ですが、
クラウドやオンプレ問わず、RDPで他サーバから接続する要件があるサーバに当てはまる内容だと思います。

# 目次

<!-- タイトルとアンカー名を編集 -->

1. [アクセス制御の種類について](#1-アクセス制御の種類について)
2. [デフォルトの設定について](#2-デフォルトの設定について)
3. [設定の復元について](#3-設定の復元について)
4. [やらかした場合の対処方法](#4-やらかした場合の対処方法)
5. [最後に](#5-最後に)

<!-- 各チャプター -->

<a id="#Chapter1"></a>

# 1-アクセス制御の種類について

まず EC2 のアクセス制御の種類についてですが、冒頭でも触れたとおり、
大まかに以下の制御方法があります。

| 名称                         | 制御対象 　   　| 制御方法    |
|:---------------------------- |:---------------|:------------|
| セキュリティグループ          | EC2             | 許可        |
| ネットワークACL               | サブネット      | 許可/拒否   |
| WAF                          | ALB、CloudFront | 許可/拒否   |
| パーソナルファイヤーウォール   | EC2            | 許可/拒否    |
- [ネットワークACL-公式ドキュメント](https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/security-groups.html)
- [セキュリティグループ-公式ドキュメント](https://docs.aws.amazon.com/ja_jp/vpc/latest/userguide/vpc-network-acls.html)
- [WAF-公式ドキュメント](https://docs.aws.amazon.com/ja_jp/waf/latest/developerguide/waf-chapter.html)

この中で WindowsFireWall はパーソナルファイヤーウォールに該当します。

<a id="#Chapter2"></a>

# 2-デフォルトの設定について

実はAWS公式が提供しているWindowsはEC2作成時点でWindowsFireWallが有効化されており、
デフォルト設定として`外部からのRDP接続をすべて許可する`設定となっています。
この設定が入ってないとRDP接続出来ないので当然ではありますが...。

#### WindowsFireWall デフォルト設定値 抜粋
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/90ca8f9c-f621-aa0b-85d1-6271bf60e406.png)


<a id="#Chapter3"></a>

# 3-設定の復元について

上記のデフォルト設定についてWindowsFireWallが無効化されている状態限定ですが、
ワンクリックで`すべての設定を削除する`という、自爆スイッチが存在します。
以下画面の赤枠のボタン「設定の復元」がその自爆スイッチです...。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/86fb2383-16a6-a283-807d-929d932a5e70.png)

これをクリックすると数秒後に接続が切断され...。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/c39480a6-795c-12ca-aca9-91398b99eda6.png)

そして永遠に繋がらなくなります...。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/a2f298cc-5787-3b02-09e1-2500c05b720e.png)

以降は RDPはおろか、 Ping や TraceRoute すら通らない状態になります。

<a id="#Chapter4"></a>

# 4-やらかした場合の対処方法

もしこれをやらかしてしまった場合、以下の方法で対処が可能です。

### (1)フリートマネージャー(Session Manager) で接続して WindowsFireWall を無効化

フリートマネージャーでの接続は WindowsFireWall で弾かれないため、
ログインしてWindowsFireWall を無効化しましょう。

ただしフリートマネージャーを利用するための前提として以下の設定が必要となります。
なおSSMエージェントは`Windows Server 2016以降`は標準でインストールされているので、
基本的にはIAM権限の設定やVPCエンドポイントを作成すれば利用できる場合が多いかと思います。

```text:前提条件
・SSMエージェントがインストールされていること
・インスタンスプロファイルでSSM接続のIAM権限の付与されていること
・VPCにSSM用のVPCエンドポイントが作成されていること　(Privateサブネット限定)
・セキュリティーグループのアウトバウンドルールで、
　VPCエンドポイント宛の443ポートが許可されていること　(Privateサブネット限定)
```

詳しい前提条件や設定は公式ドキュメントにてご参照ください。
[Session Manager の前提条件 - 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/session-manager-prerequisites.html)

### (2)SSM Run Command で WindowsFireWall を無効化

(1)と同様に、SSM 関連サービスである、Run Command を利用した方法です。
AWS公式が配布している Run Command に WindowsFireWall 無効化できるドキュメントがあり、
それを対象のWindowsに対して実行します。

[AWSSupport-TroubleshootRDP - 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/systems-manager-automation-runbooks/latest/userguide/automation-awssupport-troubleshootrdp.html)


ただしこの方法もSSMエージェントを利用しているため、
(1)と同様の事前設定が必要となります。

### (3)リモートレジストリを利用して WindowsFireWall を無効化

接続不可になった Windows の ルートボリュームをデタッチし、
他の Windows に EC2 へアタッチしてレジストリを変更することでWindowsFireWallを無効化できます。
[リモートレジストリを使用 - 公式ドキュメント](https://docs.aws.amazon.com/ja_jp/AWSEC2/latest/WindowsGuide/troubleshoot-connect-windows-instance.html#disable-firewall)

SSMエージェントを利用できない場合はこの方法が有効となります。

### (4)EC2 のリストア

最終手段ですが、素直にAMI からリストアします。
設定作業の出戻りは発生しますが、一番手堅い方法でもあります。

<a id="#Chapter5"></a>

# 5-最後に
私がやらかした当時はSSMエージェントが利用できたのですぐに復旧ができましたが、
最悪リストアもありえたかなりデンジャーな事件でした。

皆様もやらかしてしまった場合は本記事を参考に対処いただければ幸いです。