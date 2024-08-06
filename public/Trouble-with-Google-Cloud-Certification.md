---
title: Google Cloud 認定試験 リモート試験でのトラブルと対処方法について
tags:
  - 'Google'
  - 'GoogleCloud'
  - '資格'
  - '認定試験'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- 発端や概要を記載 -->
# はじめに

直近1ヶ月で複数のGoogle Cloud 認定試験をテストセンターで受けていたのですが、
最近の猛暑が厳しすぎる影響でリモート試験を受けてみました。
Google Cloudに限らずリモート試験自体が初めてだったため入念に準備しましたが、当日いくつかトラブルに見舞われました。
~~今思えば`ホームグラウンドだし余裕っしょ！`という慢心があったのかもしれません...。~~

という訳で具体的な試験の流れと遭遇したトラブル、そしてその対処方法を記載します。
本記事を見た方が同じ悲劇に見舞われないことを願っています...。

なお本記事はリモート試験の話にフォーカスしています。
なので`そもそもGoogle Cloud 認定試験て何なんだい？`という方は以下ドキュメントを覗いてみてください。

* [Google Cloud 認定資格](https://cloud.google.com/learn/certification?hl=ja)
* [Google Cloud 認定資格の一覧を解説。全部で何個ある？難易度は？](https://blog.g-gen.co.jp/entry/google-cloud-certification)


<!-- タイトルとアンカー名を編集 -->
# 目次
1. [事前準備](#1事前準備)
2. [試験当日の流れ](#2試験当日の流れ)
3. [トラブルと対処方法](#3トラブルと対処方法)
4. [最後に](#4最後に)


<!-- 各チャプター -->
<a id="#Chapter1"></a>

# 1.事前準備

### (1).試験の予約
Google Cloud 認定試験はwebassessorにて予約します、
アカウントを持っていない場合は作成して各種設定をしてください。
なおテストセンターでの試験に比べると枠は空いてることが多いですが、早めの予約をおすすめします。

* [webassessor](https://www.webassessor.com/)

### (2).セキュアブラウザのインストール
試験予約ページからセキュアブラウザをインストールしてください、
詳しい手順を以下をご確認ください。

* [How to Install the Respondus Secure Browser](https://kryterion.my.site.com/support/s/article/How-to-Install-the-Respondus-LockDown-Browser?language=en_US)

### (3).生体認証の登録
こちらも試験予約ページから登録を実施してください、
詳しい手順を以下をご確認ください。

* [Create A Biometric Profile / Enroll in Biometrics](https://kryterion.my.site.com/support/s/article/Creating-your-Biometric-Profile?language=en_US)

### (4).システム要件・試験環境の確認
試験を受けるためのPCのシステム要件は以下にてご確認ください、
一般的なノートPC(Windows or Mac)であれば特に問題は無いかと思います。

* [Online Testing Requirements](https://kryterion.my.site.com/support/s/article/Online-Testing-Requirements?language=en_US)
* [Camera Requirements](https://kryterion.my.site.com/support/s/article/What-Cameras-and-Camera-Settings-are-Required-for-an-Online-Proctored-OLP-Exam?language=en_US)

また試験環境の要件は以下をご確認ください。
色々書いてありますが少なくとも、`利用可能なデバイス`、`試験環境の整理整頓`は意識することをおすすめします。

* [Testing Environment Requirements](https://kryterion.my.site.com/support/s/article/Launching-your-Online-exam?language=en_US#:~:text=%E3%81%8F%E3%81%A0%E3%81%95%E3%81%84%E3%80%82-,%E3%83%86%E3%82%B9%E3%83%88%E7%92%B0%E5%A2%83%E3%81%AE%E8%A6%81%E4%BB%B6,-%E3%81%8A%E9%83%A8%E5%B1%8B%E3%81%AF%E6%98%8E)


<a id="#Chapter2"></a>

# 2.試験当日の流れ

### (1).セルフチェック
試験を開始する前にもう一度、[事前準備](#1事前準備)の内容をチェックすることをおすすめします。

また試験を受けるPCにて`余計なプロセスが立ち上がっていないこと`をご確認ください、
特にスタートアップ時に自動で立ち上がるプロセスは見落としがちなので、事前に停止しておくと後々スムーズに進みます。

なお必須ではないですが以下に`ウイルス対策ソフトウェアの無効化が必要な場合がある`、と記載があったため、
筆者はWindows Defenderを無効にしました。
(無効化した場合は後で有効化することをお忘れなく...。)

* [Additional Considerations](https://kryterion.my.site.com/support/s/article/Online-Testing-Requirements?language=en_US#:~:text=Additional%20Considerations%3A)

### (2).事前チェック
開始時間の10分前になるとwebassessorの試験予約ページにて、受ける試験の`?`のマークが`試験開始`ボタンに変わり、
クリックすることでセキュアブラウザが自動的に起動します。
ここで事前チェックとして、`生体認証`、`受験用PCのカメラ/マイクの確認`、`写真付き身分証明書の撮影`、`試験環境の撮影`を実施します。
詳しい内容は以下ドキュメントにてご確認ください。

* [事前チェックのプロセス](https://kryterion.my.site.com/support/s/article/Launching-your-Online-exam?language=en_US#:~:text=%E3%81%8F%E3%81%A0%E3%81%95%E3%81%84%E3%80%82-,%E4%BA%8B%E5%89%8D%E3%83%81%E3%82%A7%E3%83%83%E3%82%AF%E3%81%AE%E3%83%97%E3%83%AD%E3%82%BB%E3%82%B9,-%E8%A9%A6%E9%A8%93%E4%B8%BB%E5%82%AC%E8%80%85)

### (3).試験官との環境チェック・試験の説明
ここではチャット形式で試験官とのやり取りが発生します。
事前チェックで提出した内容に不備/不足がある場合は、追加で身分証明書の撮影や試験環境の撮影を依頼され、
それが終われば試験の説明がされます。

最初は英語でやり取りが始まりますが依頼すれば日本語での対応してもらうことも可能で、
またチャットのみで完結したので、あまり緊張しなくて大丈夫かと思います。

### (4).試験開始
以降は試験会場で受けるのと同じ画面が表示されます。
試験を受けるための同意書にチェックして、試験を開始します。

### (5).試験終了
回答が終わったら試験完了ボタンをクリックして終了します。
テストセンターの試験と同様に、試験完了画面に合否(Pass or Fail)が表示されます。


<a id="#Chapter3"></a>

# 3.トラブルと対処方法

### セキュアブラウザが立ち上がらない
* 事象
[(2).事前チェック](#2事前チェック)にて`試験開始`ボタンをクリック後、
試験の妨げになると判断されるプロセスがあり、セキュアブラウザのポップアップにて対象の停止を促される。

* 対処方法
ポップアップにて`○○プロセスを停止してください`のような表示がされるので、
停止して問題なければポップアップ左下の停止ボタンをクリックしてください。
これは停止が必要なプロセスの数だけ繰り返す必要があります。

### 試験環境の撮影が通らない
* 事象
[(3).試験官との環境チェック・試験の説明](#3試験官との環境チェック試験の説明)にて、試験環境の撮影のチェックが中々通らない。

* 対処方法
対処とは言い難いですが、指示された場所の撮影を根気強く続けてチェックを通りました。
原因は`ノートPCのカメラの画質が悪く不明瞭な映像になっていた`こと、
また筆者の`試験環境が散らかっていたこと`だと思われます...。

そのため対策として、`画質が良い外付けカメラがあれば使用すること`、
`試験環境をしっかり片づけてから試験を受けること`をおすすめします。

### モニターを接続したらセキュアブラウザが終了した
* 事象
[(3).試験官との環境チェック・試験の説明](#3試験官との環境チェック試験の説明)にて、試験環境の撮影後にモニターを再度接続したところ、
セキュアブラウザが終了した。

* 対処方法
セキュアブラウザ終了後に再度立ち上げましたが、[(2).事前チェック](#2事前チェック)から再開となり、
その時点で試験開始時間を20分ほど超過していたため、再度受験ができなくなってしまいました。

サポートケースには似たような事例はなく、またメール問い合わせは時間がかかると判断し、
泣く泣く予約ページにて15分後にリスケジュールして試験を受けました。
(そして試験当日のリスケジュールは追加料金がかかりました...。)

* [Contact Support](https://kryterion.my.site.com/support/s/contactsupport?language=en_US)
* [reschedule assessment](https://kryterion.my.site.com/support/s/article/How-to-reschedule-cancel-your-assessment?language=en_US)

憶測ではありますが、`セキュアブラウザを立ち上げた後に、モニターを接続したこと`自体がセキュアブラウザ終了の原因と考えられます。
以下記載の通り利用できるデバイスの数に制限があることから、
`再接続したことで別のモニターとしてカウントされた`、または`セキュアブラウザ立ち上げ後の接続自体がNG`だと思われます。
正直試験内容よりもこのトラブルが苦しかったので、皆様はくれぐれもお気をつけてください...。

```注釈抜粋
Your immediate surroundings are clutter-free.
There is only one active computer, one active monitor, one keyboard, and one mouse.
```

* [Testing Environment Requirements](https://kryterion.my.site.com/support/s/article/Launching-your-Online-exam?language=en_US#:~:text=%E3%81%8F%E3%81%A0%E3%81%95%E3%81%84%E3%80%82-,%E3%83%86%E3%82%B9%E3%83%88%E7%92%B0%E5%A2%83%E3%81%AE%E8%A6%81%E4%BB%B6,-%E3%81%8A%E9%83%A8%E5%B1%8B%E3%81%AF%E6%98%8E)


<a id="#Chapter4"></a>

# 4.最後に
本記事を読まれた方の大半がお気づきかと思いますが、`事前にしっかりと試験環境を準備すること`が大事です。
どうしても試験自体に注意が向きがちかと思いますが、
リモート試験の場合はそれ以外にも注意すべきことが多いので、これから試験を受ける方はお気を付けください。

~~(筆者は片付けと部屋の撮影がめんどくさいので、テストセンターに赴くスタイルに戻ります...。)~~
