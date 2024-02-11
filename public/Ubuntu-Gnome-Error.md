---
title: Gnome(gdm3)がお亡くなりになった場合の対処方法について
tags:
  - 'Ubuntu'
  - 'gnome'
  - 'GUI'
  - 'Desktop'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- 発端や概要を記載 -->
# はじめに

先日自宅で愛用しているUbuntuのgdm3だけ唐突にお亡くなりになる事象が発生しました。
復旧にあたり色々試してみた結果とりあえず解消したので、今回はその対処方法について共有します。

<!-- 各チャプター -->
<a id="#Chapter1"></a>
# 1.実行環境

環境は以下の通りです。
また最後にアップグレード(apt update & apt upgrade)したのは、
Ubuntu Desktopをインストールした時です。

| 項目                | バージョン          |
|:--------------------|:--------------------|
|	OS                  | Ubuntu 23.04        |
|	Gnome Shell         | GNOME Shell 44.3    |
|	gdm3                | GDM 44.0            |


<a id="#Chapter2"></a>
# 2.何が起きたか

時刻は午前0時。
最近日課のパ〇ワールドを終え深夜テンションに入ったところで、
k8sのワーカーノード用に買ったまま放置していたミニPCのことを思い出しました。

そしてセットアップのためk8sのマスタノードのUbuntuへリモデしたところ、何故かつながらない。
![gdm3-リモデエラー画面.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/5a1c0c0d-9b21-029e-9d44-b83e9560c91c.png)

幸いSSHは繋がったのでログを調べてみると、何故かgdm3だけがお亡くなりになっていました。
しかもコアダンプが出力されており、明らかに良からぬ事が起こっているという状況でした。
```text:エラーログ抜粋 (journalctl -u gdm.service)
kernel: traps: gdm3[897] trap int3 ip:7f83d1bb8561 sp:7ffc702f9d60 error:0 in libglib-2.0.so.0.7600.4[7f83d1b77000+9a000]
systemd[1]: gdm.service: Failed with result 'core-dump'.
```
ぱっと見だとlibglib-2.0.so.0.7600.4というライブラリで問題が発生しているようですが、
最近アップグレードした記憶がないから心当たりがない...。
こうして私は自宅でも障害対応に追われることとなりました。

なお原因調査にあたりArch LinuxのWikiを参考にしました。
ここはArch Linuxに限らず大抵のLinuxディストリビューションで役立つので、個人的によく参考にしています。

- [GDM Wiki](https://wiki.archlinux.jp/index.php/GDM)
- [GNOMEトラブルシューティング Wiki](https://wiki.archlinux.jp/index.php/GNOME/%E3%83%88%E3%83%A9%E3%83%96%E3%83%AB%E3%82%B7%E3%83%A5%E3%83%BC%E3%83%86%E3%82%A3%E3%83%B3%E3%82%B0)


<a id="#Chapter3"></a>

# 3.対処したこと

### (1) OS再起動 & gdm3再起動
とにもかくにもまずはOS再起動/プロセス再起動です。
オンプレ、クラウド限らず大抵のことは再起動すれば治ると私は信じています。

```text:OS再起動
sudo systemctl reboot
```
```text:gdm3再起動
sudo systemctl restart gdm3.service
```

しかし結果は以下の通り、状況変わらず同様のエラーが発生していました。
```text:エラーログ抜粋
kernel: traps: gdm3[897] trap int3 ip:7f83d1bb8561 sp:7ffc702f9d60 error:0 in libglib-2.0.so.0.7600.4[7f83d1b77000+9a000]
systemd[1]: gdm.service: Failed with result 'core-dump'.
systemd[1]: Failed to start gdm.service - GNOME Display Manager.
systemd[1]: gdm.service: Triggering OnFailure= dependencies.
```

### (2) gdm3の設定ファイルを再構築
何らかの理由で設定ファイルで問題が発生している可能性があるので、
以下コマンドでgdm3の設定を再構成して再起動してみました。
```text:再設定コマンド
sudo dpkg-reconfigure gdm3
sudo systemctl reboot
```
しかし状況は変わらず...。


### (3) Wayland無効化
実はまだWaylandに対応していないソフトウェアがあるということで、
そういった場合はWaylandではなく従来のXorgを利用することが推奨されているようです。
なのでgdm3の設定ファイル(custom.conf)を編集して、Waylandを使用できないようにします。

以下のコメントアウトされた行があるので、#を外してWaylandを使用しないようにします。
```diff_shell:ファイル編集 (/etc/gdm3/custom.conf)
- #WaylandEnable=false
+ WaylandEnable=false
```
しかしこれでも状況は変わらず...。


### (4) gdm3のインストール
もはや最終手段ですが、gdm3を再インストールして再起動してみました。
```text:コマンド
sudo apt-get purge gdm3
sudo apt-get install gdm3
sudo systemctl reboot
```

結果としてデスクトップ環境が復活しました。
```text:ログ抜粋
systemd[1]: Starting gdm.service - GNOME Display Manager...
systemd[1]: Started gdm.service - GNOME Display Manager.
gdm-launch-environment][921]: pam_unix(gdm-launch-environment:session): session opened for user gdm(uid=120) by (uid=0)
```

これで治らなかったらコアダンプを解析するか、最悪OS再インストールぐらいしかないかなぁ、
と思っていたので正直助かりました。

<a id="#Chapter4"></a>

# 4.最後に
結局のところ詳細な原因はわからず、
再インストールで治すというかなり力技になってしました...。

ただ調べていく中で本記事と似たような事象がUbuntuの様々なバージョンで発生しているようで、
もしかしたらgnomeあるあるなのかもしれません。
なのでもし同様の事象が起きた際は参考にしていただければ幸いです。