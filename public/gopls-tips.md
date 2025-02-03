---
title: 荒ぶるgoplsよ！！鎮まり給え！！
tags:
  - Go
  - VSCode
  - gopls
private: false
updated_at: '2025-02-03T12:55:31+09:00'
id: be9f2e3e2456dbf88de2
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- 発端や概要を記載 -->
# はじめに
Go言語での開発において、
gopls(Goの言語サーバー)は非常に便利な補完やリファクタリング機能を提供してくれます。
しかし、この度goplsが暴走してCPU使用率が地獄絵図になる事象を確認したため、
本記事では、そんな`「荒ぶる goplsを鎮める方法」`を紹介します。

なお筆者は以下環境で事象を確認していますが、
その他も環境でも同様の問題が起こった際はご参考にしていただければ幸いです。

* OS - Ubuntu 22.04.4 LTS (WSL2)
* VSCode - 1.96.2
* Go - 1.22.2
* gopls - [Go(VSCode拡張機能)](https://marketplace.visualstudio.com/items?itemName=golang.Go)で導入

<!-- 各チャプター -->
<a id="#Chapter1"></a>

# ➀症状

### 1. goplsのCPU使用率が異常に高い
Goのファイルを弄るときだけやけに重くなるなぁ、
と思いtopやhtopで確認すると、goplsのCPU使用率がとんでもないことになっていました。

![gopls-cpu-top.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/4d85946e-80f1-00bc-2433-cca384fe8d86.png)

同じプロセスがこれだけ立ち上げることはあまりないので、しばらく眺めていました。
![gopls-cpu-htop.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/30d794b0-fc53-ec5a-1cc1-3a095152f0b2.png)


### 2. permission denied エラーが大量発生
goplsのログ(VSCodeの出力)を見てみると、以下のようなエラーが大量出ていました。
```
[Info  - 6:38:03 PM] 2025/01/13 18:38:03 open /mnt/wslg/distro/dev/fd/7/mnt/c/Documents and Settings/All Users/Microsoft/Windows/WER/ReportArchive/NonCritical_Update;ScanForUp_fcd1f84b8840165b971819dfbdaca46b6cff5de_00000000_bb3b3e87-6021-4137-9060-5219e9d403ac: permission denied
[Info  - 6:38:03 PM] 2025/01/13 18:38:03 open /mnt/wslg/distro/dev/fd/7/mnt/c/Documents and Settings/All Users/Microsoft/Windows/WER/ReportArchive/NonCritical_Update;ScanForUp_fcd1f84b8840165b971819dfbdaca46b6cff5de_00000000_7d580537-feff-40fe-9089-cc03cfe77f38: permission denied
[Info  - 6:38:03 PM] 2025/01/13 18:38:03 open /mnt/wslg/distro/dev/fd/7/mnt/c/Documents and Settings/All Users/Microsoft/Windows/WER/Temp
8c469073-fa02-49a7-ac11-560d1e439d61: permission denied
[Info  - 6:38:03 PM] 2025/01/13 18:38:03 open /mnt/wslg/distro/dev/fd/7/mnt/c/Documents and Settings/All Users/Microsoft/Windows/wfp: permission denied
[Info  - 6:38:03 PM] 2025/01/13 18:38:03 open /proc/self/task/2016225/fd/7/mnt/wslg/distro/dev/fd/8/c/Documents and Settings/kanik/AppData/Local/Application Data/ElevatedDiagnostics: permission denied
.
.
.
[Info  - 6:42:31 PM] 2025/01/13 18:42:31 open /proc/self/task/2016225/fd/7/mnt/wslg/distro/dev/fd/8/c/Windows/System32/networklist: permission denied
[Info  - 6:42:32 PM] 2025/01/13 18:42:32 open /proc/self/task/2016225/fd/7/mnt/wslg/distro/dev/fd/8/c/Windows/System32/SleepStudy: permission denied
[Info  - 6:42:32 PM] 2025/01/13 18:42:32 open /proc/self/task/2016225/fd/7/mnt/wslg/distro/dev/fd/8/c/Windows/System32/spool/PRINTERS: permission denied
[Info  - 6:42:32 PM] 2025/01/13 18:42:32 open /proc/self/task/2016225/fd/7/mnt/wslg/distro/dev/fd/8/c/Windows/System32/spool/SERVERS: permission denied
[Info  - 6:42:32 PM] 2025/01/13 18:42:32 open /proc/self/task/2016225/fd/7/mnt/wslg/distro/dev/fd/8/c/Windows/System32/sru: permission denied
[Info  - 6:42:32 PM] 2025/01/13 18:42:32 open /proc/self/task/2016225/fd/7/mnt/wslg/distro/dev/fd/8/c/Windows/System32/Tasks: permission denied
```

はい、見ての通りですが、
`goplsがルートディレクトリ配下をスキャンしようとして権限不足でエラー`を吐いている模様です…。


<a id="#Chapter2"></a>

# ➁原因と対策

## 原因： ルートディレクトリ直下にファイルが置かれている
本事象の直接的な原因は、
`「ルートディレクトリ直下にGo関連ファイルが置かれていた」`ことです。

どういうことかと言うと、goplsのワークスペースの仕様が影響しています。
[gopls, the Go language server](https://github.com/golang/tools/blob/master/gopls/README.md)
[Gopls: Setting up your workspace](https://github.com/golang/tools/blob/master/gopls/doc/workspace.md)

### gopls のワークスペースとは
ワークスペースとは`gopls が認識する Go コードの集合`であり、
言語サーバーとしての機能(コード補完、参照検索、リネームなど)を提供する対象範囲を指します。

通常、`「どのディレクトリ(フォルダ)をエディタ/IDE が開いているか」`に基づいて、このワークスペースが構築されます。

### ワークスペースの決まり方

VS Code 上で gopls を利用するとき、どのディレクトリが gopls のワークスペースとして認識されるかは主に以下の要素によって決まります。

#### (1).VS Code 上で開いているフォルダ

- VS Code で開いているフォルダが gopls のワークスペースとして認識される

#### (2).go.work / go.mod / GOPATH の優先度

- gopls は「Go モジュール (go.mod)」や「Go ワークスペース (go.work)」のファイルを検出すると、それを基準にワークスペースを構成する
- VS Code で開いているフォルダ以下に「go.work」があれば、そこに定義された複数のモジュールをまとめて認識する
- 「go.mod」がある場合は、そのモジュール単位でワークスペースが認識される
- いずれもない場合は、GOPATH モードとして扱われることがあるが、Go 1.18 以降は基本的にモジュール運用が主流

---

今回のケースでは、
`ルートディレクトリ直下に go.mod、go.sum、そして .git などが配置`されていたため、
そこを起点としてgoplsの処理が走ってしまったようです。

なお、ルートディレクトリに誤ってファイルを置いていた理由については、
おそらくDocker Composeのバインドマウント設定が誤っており、その結果作成されたものが放置されていたのではないかと推測しています……。


## 解消方法：ルートディレクトリ直下のファイルを削除
以下のようなGo関連ファイルがルートディレクトリに配置されている場合は削除して、
今後は配置しないようにしてください。

- XXX.go
- go.mod
- go.sum
- .git

## 対策：スキャン対象を制限する
次に対策ですが、
gopls の設定ファイルにて`ワークスペースでのスキャン対象となるディレクトリを指定`できます。
今回はVS Codeに導入しているgoplsを例に挙げているため、
VS Codeの設定ファイル(settings.json)を修正することで同様の事象を防ぐことが可能です。

* gopls のスキャン対象を制限
VS Code の設定ファイル(settings.json)で、directoryFilters を追加することでスキャン対象を絞り込んだり、特定のディレクトリを除外したりできます。

[build.directoryFilters](https://github.com/golang/vscode-go/blob/master/docs/settings.md#builddirectoryfilters)

以下設定は一例なので、
実際に事象が発生した場合はエラーログを元に必要に応じて設定してください。

```json:settings.json
{
  "gopls": {
    "verboseOutput": false,
    "directoryFilters": [
      "-node_modules",
      "-vendor",
      "-/mnt/",
      "-/proc/"
    ]
  }
}
```

あらかじめ除外設定をしておけば、
今回のような問題を未然に防ぐことができますので、ぜひ設定を行ってみてください。


<a id="#Chapter2"></a>

# おわりに

ここまで読んでいただきありがとうございます。
エラー原因がかなり初歩的な事で正直恥ずかしいですが、
もしgoplsが荒ぶることがあれば参考にしていただけますと幸いです。

皆さんの Go 開発ライフに平穏が訪れますよう、心より願っています。
それでは――
**荒ぶるgoplsよ！！鎮まり給え！！**
