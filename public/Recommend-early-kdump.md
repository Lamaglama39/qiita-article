---
title: early kdumpのすゝめ
tags:
  - RHEL
  - kdump
private: false
updated_at: '2023-11-09T00:24:16+09:00'
id: c66dc8173fee4c92f13b
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに
<!-- 発端や概要を記載 -->
あなたは`early kdump`をご存じですか？
恥ずかしながら私はつい先日、カーネルパニック起因の障害調査をした際に存在を知りました...。

調べるうちにearly kdumpはサーバ構築の標準設定にするべきとわかりましたが、
まともに紹介している記事がRHEL公式ドキュメントぐらいしかなかったため、周知も含めて本記事でおすすめさせてください。

# 目次
<!-- タイトルとアンカー名を編集 -->
1. [early-kdumpとは](#1-early-kdumpとは)
2. [設定すると何が嬉しいか](#2-設定すると何が嬉しいか)
3. [設定方法](#3-設定方法)
4. [参考文献](#4-参考文献)

<!-- 各チャプター -->
<a id="#Chapter1"></a>

# 1-early-kdumpとは
「early kdump」は、kdumpの一部として実装されている機能です。
名前通りサービス起動までの時間が重視された機能であり、
`OS起動段階でカーネルクラッシュが発生した場合のクラッシュダンプ取得`を目的に、
RHEL8から導入された比較的新しめの機能です。

```text:early kdumpについて
kdump サービスが起動していないと、起動段階でカーネルがクラッシュし、クラッシュしたカーネルメモリーの内容を取得して保存できません。
そのため、トラブルシューティングの重要な情報は失われます。

この問題を対処するために、RHEL 8 では、early kdump 機能が kdump サービスの一部として導入されました。
```

[early kdumpについて-公式ドキュメント](https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/using-early-kdump-to-capture-boot-time-crashes_managing-monitoring-and-updating-the-kernel#doc-wrapper)

<a id="#Chapter2"></a>

# 2-設定すると何が嬉しいか
前述の通りkdumpサービスの一機能のため、サービス起動までの時間が早いこと以外の違いはありません。
しかしこの起動時間がかなり重要で、以下条件で確認した限りでは`10倍`の差があります。

### 検証条件
| 項目               | 条件     |
|:-------------------|:---------|
| 実行環境           | EC2      |
| インスタンスタイプ | t2.micro |
| OS                | RHEL 9.2 |

### 検証結果
```text:kdumpの場合
(1)サーバ起動時間
Oct 29 14:25:19 ip-10-0-1-10 kernel: The list of certified hardware and cloud instances for Red Hat Enterprise Linux 9 can be viewed at the Red Hat Ecosystem Catalog, https://catalog.redhat.com.

(2)kdump起動時間
Oct 29 14:25:29 ip-10-0-1-10 kdumpctl[939]: kdump: kexec: loaded kdump kernel

⇒kdump起動まで、約10秒
```

```text:early kdumpの場合
(1)サーバ起動時間
Oct 29 14:38:59 ip-10-0-1-10 kernel: The list of certified hardware and cloud instances for Red Hat Enterprise Linux 9 can be viewed at the Red Hat Ecosystem Catalog, https://catalog.redhat.com.

(2)early-kdump起動時間
Oct 29 14:39:00 ip-10-0-1-10 dracut-cmdline[247]: kdump: kexec: loaded early-kdump kernel

⇒early-kdump起動まで、約1秒
```

上記はあくまで一例のため、サーバ構成によって短縮される時間はまちまちです。
しかし`kdump起動前にカーネルパニックが発生して、クラッシュダンプ(/var/crash)が取れなかった...。`
という事態を可能な限り避けるためにも、標準設定としてサーバ構築時に設定するべきかと思います。

<a id="#Chapter3"></a>

# 3-設定方法
基本的には公式ドキュメント記載の手順で問題なく設定できます。
[early kdumpの有効化-公式ドキュメント](https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/enabling-early-kdump_using-early-kdump-to-capture-boot-time-crashes)

1. まずkdumpが有効でアクティブであることを確認します。
そもそもkdumpが無効化されている、またはインストールされていない場合は以下手順で有効化してください。
[kdumpインストール-公式ドキュメント](https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/installing-kdump-command-lineinstalling-kdump)
``````bash
$ systemctl is-enabled kdump.service && systemctl is-active kdump.service
enabled
active
``````

2. 起動カーネルの initramfs イメージを、early kdump 機能で再構築してください。
``````bash
$ dracut -f --add earlykdump
``````

3. rd.earlykdump カーネルコマンドラインパラメーターを追加してください。
``````bash
$ grubby --update-kernel=/boot/vmlinuz-$(uname -r) --args="rd.earlykdump"
``````

4. rd.earlykdump が正常に追加され、early kdump 機能が有効になっていることを確認します。
cmdlineに`rd.earlykdump`が含まれており、
journalctlで`early-kdump is enabled.`,`loaded early-kdump kernel`が含まれていれば、有効化されています。
``````bash
$ cat /proc/cmdline
BOOT_IMAGE=(hd0,msdos1)/vmlinuz-4.18.0-187.el8.x86_64 root=/dev/mapper/rhel-root ro crashkernel=auto resume=/dev/mapper/rhel-swap rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet rd.earlykdump

$ journalctl -x | grep early-kdump
Mar 20 15:44:41 redhat dracut-cmdline[304]: early-kdump is enabled.
Mar 20 15:44:42 redhat dracut-cmdline[304]: kexec: loaded early-kdump kernel
``````

また実行環境がEC2の場合はユーザーデータに設定することで、インスタンス作成段階で有効化することをおすすめします。
```bash:ユーザーデータ
#!/bin/bash
sudo dracut -f --add earlykdump
sudo grubby --update-kernel=/boot/vmlinuz-$(uname -r) --args="rd.earlykdump"
sudo systemctl reboot
```

<a id="#reference"></a>

## 4-参考文献

* [early kdumpについて-公式ドキュメント](https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/using-early-kdump-to-capture-boot-time-crashes_managing-monitoring-and-updating-the-kernel#doc-wrapper)
* [early kdumpの有効化-公式ドキュメント](https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/enabling-early-kdump_using-early-kdump-to-capture-boot-time-crashes)
* [kdumpインストール-公式ドキュメント](https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/installing-kdump-command-lineinstalling-kdump)
