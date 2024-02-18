---
title: cool retro termが良すぎるので紹介したい。
tags:
  - 'Linux'
  - 'Terminal'
  - 'CUI'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- 発端や概要を記載 -->
# はじめに

皆さんはどのターミナルエミュレータを利用していますか？
GNOME Terminal、Konsole、Xfce Terminal、Terminator、Tilix、Guake、Yakuake、Alacritty、RXVT-Unicode、Tmux...。
挙げていけばきりがないですが、個人的には`cool retro term`がおすすめです。

このターミナルエミュレータの魅力は、何といってもビジュアルにあります。

![retro-橙画面-01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/d71073af-f1a5-2cd5-2392-02f4e407b5e9.png)
~~私のスクショだと魅力が伝わりきらないので、~~
~~よろしければ公式GithubのGIFをご覧ください...。~~
- [Github cool-retro-term](https://github.com/Swordfish90/cool-retro-term)

これはブラウン管モニターを彷彿とさせるレトロなビジュアルを再現しており、古いモニター特有のビジュアルエフェクトを色々設定できるので、
レトロな雰囲気が好きな人や、日常のターミナル作業の遊び心に飢えている人に最適です。

ということで前説が長くなってしましたが本記事では導入方法含め、設定方法、問題点を含めもろもろ紹介します。

<!-- 各チャプター -->
<a id="#Chapter1"></a>
# 1.インストール方法

## Linux
まずUbuntu、Debianについては公式でパッケージ化されており、
以下のコマンドでインストールできます。
```text:Ubuntu/Debian インストールコマンド
sudo apt -y update && sudo apt -y upgrade
sudo apt install -y cool-retro-term
```

その他のディストリビューションは公式でパッケージ化されていたりいなかったりするので、
snap store経由でのインストールがおすすめです。
具体的には以下ドキュメントにOSごとの手順が記載されているため、こちらをご確認ください。
- [snap store cool-retro-term](https://snapcraft.io/cool-retro-term)

またソースからビルドすることも可能なので、その場合は以下をご確認ください。
- [build cool-retro-term](https://github.com/Swordfish90/cool-retro-term/wiki/Build-Instructions-(Linux))

## Mac
brew caskでインストールできます。
またLinux同様に、ソースからビルドすることも可能です。
```text:Mac インストールコマンド
brew install --cask cool-retro-term
```
- [brew cask cool-retro-term](https://formulae.brew.sh/cask/cool-retro-term)

## Windows
残念ながらWindows版は用意されていません...、
なのでWindowsで使う場合は以下の二択になります。
```
(1) WSL上のLinuxで実行して、X Serverなどを経由して使う
(2) cool retro termをインストールしたLinux or Macにリモートデスクトップして使う
```
(1)のWSLで利用する方法については、以下ドキュメントをご参照ください。
- [Windows cool-retro-term](https://gist.github.com/h3r/2d5dcb2f64cf34b6f7fdad85c57c1a45)

(2)のリモートデスクトップ経由の利用はリソース消費量の観点からあまりおすすめできません...、
詳細は後述の問題点にて解説します。

<a id="#Chapter2"></a>
# 2.設定方法
`[cool-retro-termを起動]→[ターミナル画面をマウス右クリック]→[Settings]`で設定画面が開けます。
ここでフォントやコンソール画面のエフェクトなど、様々な設定ができます。
なお設定ファイルは`$HOME/.config/cool-retro-term/cool-retro-term.conf`ですが、
GUIで設定変更したほうが見た目の変化が分かりやすいのでそちらを推奨します。

![retro-設定画面-01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/7f53b7db-62db-43cf-4d4e-6e834178ec7f.png)

以下は設定の一覧です。

### (1)General
* **Profile** - デフォルトで用意されている設定
* **Screen** - ターミナル画面に関する設定
(明るさ、コントラスト、フレームサイズ、透過...etc)

### (2)Terminal
* **Cursor** - カーソルの点滅設定
* **Font** - フォント全般の設定
(フォントの種類、フォントサイズ、太さの設定)
* **Colors** - カラー設定
(フォント、コンソール画面)

### (3)Effects
* **Effects** - ターミナル画面のエフェクト設定
(ブルーム、画面の焼き付き、ノイズ、ジッター、画面の湾曲、点滅、水平同期、グリッチ...etc)

### (4)Advanced
* **Command** - 起動時のカスタマイズコマンド設定

* **Performance** - エフェクトのパフォーマンス設定
(FPS、テクスチャ/ブルーム/画面の焼き付きのクオリティ)

<a id="#Chapter3"></a>
# 3.問題点

:::note warn
以降は筆者の環境で発生した事象です。
ただしマシンスペックに依存すると思うので、あくまで参考情報として記載します。
:::

こんな素敵なcool retro termですが、一つだけ問題点があります。
それはリモートデスクトップ経由で接続して利用する場合、CPU使用率がとんでもない状態になることです。

まずこちらがUbuntuから直接実行した場合です。
![retro-実機リソース消費-01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/b37cca20-1bbd-1a4d-c75b-38f892c9b29a.png)

次にWindowsからリモートデスクトップ経由(xrdp)で実行した場合です。
![retro-リモデ経由リソース消費-01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/4e64035d-13ef-ebdb-71ee-08d32786849b.png)

ご覧の通りCPU使用率が爆増して、動作がかなりもっさりになり入出力の遅延も半端ないことになります。

推測にはなりますが、そもそもリモートデスクトップ経由の接続が重くなることに加え、
cool retro termの場合はエフェクトが豪華すぎるため、グラフィックレンダリング処理の負荷が高いことが原因ではないかと思います。

~~設定で各種エフェクトを無効化すればかなり改善されますが、ビジュアルの良さがなくなるというジレンマ...。~~