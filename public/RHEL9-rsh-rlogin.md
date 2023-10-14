---
title: RHEL9でrloginを無効化して、rshだけ有効化する方法
tags:
  - Linux
  - RHEL
  - rsh
private: false
updated_at: '2023-10-12T23:52:47+09:00'
id: c8ba23172429b0508e1f
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに
rsh.....、ずいぶん前にsshに取って代わられて、今となっては実運用で使うことは稀でしょう。
なおかつ「rloginは使えない状態で、rshだけ使えるようにする」なんて要件はまずないでしょう。

そんなレアケースに最近かかわり、若干ハマった箇所があったのでもろもろ共有します。
※引用しているドキュメントはAIXのものですが、コマンド仕様自体はRHELでも基本的に同じです

## 目次
<!-- タイトルとアンカー名を編集 -->
1. [rsh関連のサービスについて](#1-rsh関連のサービスについて)
2. [サーバー側の設定](#2-サーバー側の設定)
3. [クライアント側の設定](#3-クライアント側の設定)
4. [何故かrshでログインできない](#4-何故かrshでログインできない)
5. [結論](#5-結論) 
6. [参考文献](#6-参考文献)

<!-- 各チャプター -->
<a id="#Chapter1"></a>
## 1-rsh関連のサービスについて
`rsh-server`、`rsh`をインストールすると、以下のサービスが利用できるようになります。
この中で今回は`rsh.socket`だけ有効化して、他は無効されている状態にします。

- rsh.socket
>rsh (リモートシェル) は、ユーザーがリモートホスト上でシェルコマンドを実行するためのプロトコルです。
このサービスは、rsh コマンドによる接続要求を待ち受け、それに応じて新しいシェルセッションを開始します。

- rlogin.socket
>rlogin は、リモートホストでのログインセッションを提供します。
rloginコマンドを使ってリモートホストに接続すると、ユーザーはそのホスト上で完全なログインシェルを開始できます。
rlogin.socket サービスは rlogin コマンドによる接続要求を待ち受け、それに応じて新しいログインセッションを開始します。

- rexec.socket
>rexec (リモート実行) は、リモートホストで指定したコマンドを実行するためのプロトコルです。
このサービスは rexec コマンドによる接続要求を待ち受け、それに応じて指定されたコマンドの実行を開始します。

<a id="#Chapter2"></a>
## 2-サーバー側の設定

rhel9ではrshが標準リポジトリに含まれていないため、epelからダウンロードします。
```terminal:実行
$ sudo dnf -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
$ sudo dnf -y update
$ sudo dnf -y upgrade
```

rsh-serverをインストールして、起動しておきます。
```terminal:実行
$ sudo dnf -y --enablerepo="epel" install rsh-server
$ sudo systemctl enable rsh.socket
$ sudo systemctl start rsh.socket
```

ホスト名関連の設定をします。
本来は/etc/hosts.equivではなく~/.rhostで設定したほうがいいのですが、
設定を簡略化するためhosts.equivで設定します。
```terminal:実行
$ sudo hostnamectl set-hostname <ホスト名>
例：hostnamectl set-hostname rsh-server
```
```shell:/etc/hosts
<クライアントのプライベートIP> <クライアントのホスト名>
例：10.0.0.11 rsh-client
```
```shell:/etc/hosts.equiv
<クライアントのホスト名> <OSユーザー名>
例：rsh-client ec2-user
```

<a id="#Chapter3"></a>
## 3-クライアント側の設定

おなじくepelから、rshをインストールします。
```terminal:実行
$ sudo dnf -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
$ sudo dnf -y update
$ sudo dnf -y upgrade
$ sudo dnf -y --enablerepo="epel" install rsh
```

同じくホスト名関連の設定をします。
```terminal:実行
$ sudo hostnamectl set-hostname <ホスト名>
例：hostnamectl set-hostname rsh-client
```
```shell:/etc/hosts
<サーバのプライベートIP> <サーバのホスト名>
例：10.0.0.10 rsh-server
```


<a id="#Chapter4"></a>
## 4-何故かrshでログインできない
この状態でクライアントからrshコマンドを実行すると、以下の状態になります。
結果だけ見ると、「コマンド実行は成功しているのに、ログインだけできていない...？？」
という何とも不思議な状態になります。

```terminal:①コマンド引数なし
$ rsh <サーバのホスト名>
例：rsh rsh-server
⇒応答が帰ってこない
```
```terminal:②コマンド引数あり
$ rsh <サーバのホスト名> <コマンド>
例：rsh rsh-server ls -la
⇒正常に実行できる
```

<a id="#Chapter5"></a>
## 5-結論
結論として、rshでログインできないのは正常な状態です。
なぜならrshは以下の仕様があるからです...。
>-コマンド名が指定されている場合には、リモート・ホスト上で単一のコマンドを実行する。
 -コマンド名が指定されていない場合は、rlogin コマンドを実行する。

つまりサーバー側で`rlogin.socket`を起動していないので、rshでログインはできない、ということです。
この仕様が判明するまで私はしばらくハマってしまったため、
同じくハマっている人にこの情報が届けば幸いです。

<a id="#reference"></a>
## 6-参考文献
- [ローカル・ホストとリモート・ホストの接続](https://www.ibm.com/docs/ja/aix/7.2?topic=users-local-host-connections-remote-host)
