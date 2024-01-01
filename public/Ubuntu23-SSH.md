---
title: Ubuntu23でSSHを設定する際の注意点
tags:
  - Linux
  - Ubuntu
  - SSH
private: false
updated_at: '2024-01-01T13:47:14+09:00'
id: f498c1b4439a84a11b79
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに
<!-- 発端や概要を記載 -->
最近ミニPCを買って久しぶりにUbuntuをセットアップしたのですが、
SSHの設定でハマる箇所があったので共有します。

:::note warn
本記事ではUbunt 23.04を対象としていますが、
実際はUbuntu 22系時点での変更点となります。
:::


<!-- 各チャプター -->
<a id="#Chapter1"></a>

# 1.対象OSバージョン
本記事の対象OSは`Ubuntu 23.04`です。
選んだ理由はコードネームになっている`Lunar Lobster`ちゃんが可愛いからです。

* [Ubuntu ダウンロード](https://jp.ubuntu.com/download)
* [Ubuntu リリースリスト](https://wiki.ubuntu.com/Releases)

```text:/etc/os-release 抜粋
PRETTY_NAME="Ubuntu 23.04"
VERSION="23.04 (Lunar Lobster)"
```
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/50cde5db-ad29-6370-d79d-cc97cfc3970e.png)

<a id="#Chapter2"></a>

# 2.鍵認証で利用できる鍵について
まず鍵認証についてUbuntu 22.04の時点で、
`デフォルト設定でRSA/SHA1 暗号方式の鍵(ssh-rsa) が無効`となっており鍵認証で利用できなくなっています。

これは`Open SSH 8.8からSSH-1アルゴリズムのRSA鍵が無効`となったことに起因しており、Ubuntu 22.04では`OpenSSH_8.9p1が採用`されています。

* [Open SSH 8.8  リリースノート](https://www.openssh.com/txt/release-8.8)
* [Ubuntu 22.04 リリースノート](https://discourse.ubuntu.com/t/jammy-jellyfish-release-notes/24668)

当然Ubuntu 23.04でも以下の通り、Open SSH 8.8以降がデフォルトでインストールされています。
```text: Ubuntu 23.04のOpenSSHバージョン
ssh -V
OpenSSH_9.0p1 Ubuntu-1ubuntu8.5, OpenSSL 3.0.8 7 Feb 2023
```

そのため鍵認証を利用する場合、以下の選択肢があります。
```
1.RSA/SHA1 暗号鍵以外を利用する (公式推奨)
2.RSA/SHA1 暗号鍵が利用できるように許可設定を追加する
```

### (1) RSA/SHA1 暗号鍵以外を利用する (公式推奨)

新規に作成するサーバであればRSA/SHA1以外の暗号鍵を使いましょう。

代替となる暗号鍵についてはOpenSSH8.3のリリースノートにて、以下が推奨されています。

```text:推奨されている暗号鍵 (暗号化アルゴリズム)
・SHA-2アルゴリズム(rsa-sha2-256/512)
・ED25519アルゴリズム(ssh-ed25519)
・ECDSAアルゴリズム(ecdsa-sha2-nistp256/384/521)
```
```text:Open SSH 8.3  リリースノート抜粋
 * The RFC8332 RSA SHA-2 signature algorithms rsa-sha2-256/512. These
   algorithms have the advantage of using the same key type as
   "ssh-rsa" but use the safe SHA-2 hash algorithms. These have been
   supported since OpenSSH 7.2 and are already used by default if the
   client and server support them.

 * The ssh-ed25519 signature algorithm. It has been supported in
   OpenSSH since release 6.5.

 * The RFC5656 ECDSA algorithms: ecdsa-sha2-nistp256/384/521. These
   have been supported by OpenSSH since release 5.7.
```
* [Open SSH 8.3  リリースノート](https://www.openssh.com/txt/release-8.3)


暗号化アルゴリズムはssh-keygenで鍵を作成する際に、`-t`オプションで以下が指定できます。
```text:指定できる暗号化アルゴリズム
dsa | ecdsa | ecdsa-sk | ed25519 | ed25519-sk | rsa
```

例として以下では`ed25519`を指定しています。

```text:鍵作成 コマンド例
ssh-keygen -t ed25519
```

その後ssh-keygenを実行したサーバの`$HOME/.ssh/`配下に`id_ed25519`、`id_ed25519.pub`が作成されるので、
ssh-copy-idを実行して接続先のサーバに作成した公開鍵を登録すればOKです。
```text:公開鍵登録 コマンド例
ssh-copy-id -i ${PubKeyName} ${UserName}@${HostName or IPaddress}
```

### (2) RSA/SHA1 暗号鍵が利用できるように許可設定を追加する

sshdの設定ファイルで許可設定を追加することで、引き続きRSA/SHA1の暗号鍵を利用することもできます。
`既存のUbuntuサーバをアップデートしたけど、RSA/SHA1の暗号鍵はそのまま使いたい...`
といった場合はこの方法が有用です。
~~鍵を作り直してでも、RSA/SHA1以外を利用することがベストだとは思いますが...。~~

まずはsshdの設定ファイルを開いて、
```text:sshd設定ファイル
sudo vim /etc/ssh/sshd_config
```
末尾に以下を追記します。
```text:追記内容
HostKeyAlgorithms=+ssh-rsa
PubkeyAcceptedAlgorithms=+ssh-rsa
```

あとはsshdサービスを再起動すれば、引き続きRSA/SHA1の暗号鍵を利用できる状態になります。
```text:sshdの再起動
sudo systemctl restart ssh.service
```

<a id="#Chapter3"></a>

## 3.受付ポートの変更について

次にsshの受付ポートを変更する場合の注意点です。
従来であれば`/etc/ssh/sshd_config`に以下のように任意のポート番号を指定して、sshを再起動すれば受付ポートを変更できましたが、
Ubuntu 22.10からはこの方法では変更できません。
```text:従来のポート設定方法 (/etc/ssh/sshd_config)
Port 12345
```

これは`Ubuntu 22.10`から、デフォルトでSSHサーバーがsystemdのSocket-Based Activationを利用して起動されるようになったことに起因しています。
これにより`ssh.socket`の設定ファイルにて受付ポートを指定する方式となり、`/etc/ssh/sshd_config`で設定しても反映されません。
(私はこれで30分以上悩みました...。)

* [Ubuntu 22.10 リリースノート](https://discourse.ubuntu.com/t/kinetic-kudu-release-notes/27976)

`ssh.socket`の設定ファイルは、
`/etc/systemd/system/sockets.target.wants/ssh.socket`、であり
このファイルは`/lib/systemd/system/ssh.socket`へのシンボリックリンクとなっています。

なので`/lib/systemd/system/ssh.socket`内の、`ListenStream`の記述で受付ポートを指定できます。
以下の例では、デフォルトポート(22)の代わりに12345を指定しています。

```text:受付ポート指定 (/lib/systemd/system/ssh.socket)
[Socket]
ListenStream=
ListenStream=12345
```
※`ListenStream=`の空行でデフォルトポート(22)の設定を削除しています


あとはssh.socketとsshdを再起動すれば待ち受けポートが変更されます。
```text:ssh.socket再起動
sudo systemctl restart ssh.socket
sudo systemctl daemon-reload
```

<a id="#Chapter4"></a>

## 4.最後に
まさかSSHの設定でハマることになるとは思いませんでしたが、初心に帰って素直にリリースノートを確認することの大切さを再認識しました...。

なお本記事内の`記載誤り/認識齟齬`、または`こういった設定も罠だ！`みたいなものがあれば、ご連絡いただければ幸いです。
