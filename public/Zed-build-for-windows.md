---
title: Windows向け Zedビルド手順まとめ
tags:
  - Windows
  - エディタ
  - OSS
  - ZED
private: false
updated_at: '2024-11-05T12:44:39+09:00'
id: 6fcf6a1297b6ad957fa0
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- 発端や概要を記載 -->
# はじめに
2024年1月24日にOSSとなったZedですが、2024年11月時点ではWindowsに対応しておらず各自でビルドする必要があります。

ビルド自体は以下公式の通りに進めれば概ね問題ありませんでしたが、筆者は一部詰まった箇所があったので全体を通した手順をまとめておきます。
おそらく同じ個所で詰まる方はいないと思いますが、お役に立てれば幸いです。

* [Building Zed for Windows](https://github.com/zed-industries/zed/blob/main/docs/src/development/windows.md)

<!-- 各チャプター -->
<a id="#Chapter1"></a>

# 実行環境
* OS : Windows11 Home 
* Rust : 1.82.0
* Visual Studio : Visual Studio 2022 Community Edition 17.11.5
* CMaKe : 3.31.0-rc3

<a id="#Chapter2"></a>

# 手順
Windows向けのビルドのためWSL環境などではなく、`Windows上でビルドする`必要があります。
そのため手順内のコマンドは`PowerShell`や`コマンドプロンプト`で実行してください。

## (1),Rust インストール
ZedはRustで作られているので、Rustをインストールする必要があります。
以下公式からインストールしてください。

* [Rust](https://www.rust-lang.org/ja/tools/install)

wingetを利用する場合は以下でインストールできます。
```
winget install Rustlang.Rustup
```

インストールできたら念のためアップデートしておきましょう。
```
rustup update
```

## (2),Rust ツールチェイン インストール
ビルドにあたり必要となるツールチェインをインストールします。

```
rustup target add wasm32-wasi
```

## (3),Visual Studio(MSVC) インストール
Windows向けにビルドをするためにVisual Studio(MSVC)が必要となります。
`Visual Studio(IDE付き)` or `Build Tools for Visual Studio(ビルド/コンパイル機能のみ)`のどちらかをインストールしてください。

* [Visual Studio](https://visualstudio.microsoft.com/ja/downloads/)
* [Build Tools for Visual Studio](https://visualstudio.microsoft.com/ja/visual-cpp-build-tools/)

wingetを利用する場合は以下でインストールできます。
```text:Visual Studio
winget install Microsoft.VisualStudio.2022.Community
```
```text:Build Tools for Visual Studio
winget install Microsoft.VisualStudio.2022.BuildTools
```

インストール完了後、Visual Studio Installerで必要なコンポーネントを追加します。
まずは「ワークロード」の画面で、「C++によるデスクトップ開発」をチェックします。
インストールの詳細で`MSVC vXXX - VS 2022 C++ x64/84 ビルドツール`というオプションにチェックが入っていればOKです。
![msvc-workload.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/260a99db-c108-b376-eadc-7e71d87c7dbc.png)

次に「個別コンポーネント」の画面で「MSVC vXXX - VS 2022 C++ x64/84 Spectre 軽減ライブラリ」と検索します。
`(最新)`とついているものをチェックして、インストールします。
![msvc-add-component.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/9ad9f923-39b7-0dbc-b677-074b98c28d80.png)

筆者は個別コンポーネントの工程が抜けており、ビルドのタイミングで以下エラーに遭遇しました…。

<details><summary>build error</summary>

```
The following warnings were emitted during compilation:
warning: msvc_spectre_libs@0.1.2: No spectre-mitigated libs were found. Please modify the VS Installation to add these.
error: failed to run custom build command for `msvc_spectre_libs v0.1.2`
```

</details>

## (4),CMaKe インストール
CMaKe(ビルド自動化ツール)が必要なのでインストールします。

* [CMaKe](https://cmake.org/download/)

wingetを利用する場合は以下でインストールできます。
```
winget install Kitware.CMake
```

## (5),Zed ビルド

リポジトリをクローンします。
```
git clone https://github.com/zed-industries/zed.git
```

リポジトリ配下の`rust-toolchain.toml`を修正します。
Windows向けのみビルドするので`targets`を`stable-x86_64-pc-windows-msvc`に書き換えます。
```toml:rust-toolchain.toml
[toolchain]
channel = "1.81"
profile = "minimal"
components = [ "rustfmt", "clippy" ]
targets = [ "stable-x86_64-pc-windows-msvc" ]
```

以上で下準備は終わりなので、ビルドを実行します。
またビルド中はCPUとメモリをかなり食うので、事前に不要なアプリを落としておきましょう。

PCのスペックにもよりますが、10分程度かかるので気長に待ちましょう。
```
cargo build --release
```

ビルド完了後、リポジトリ配下に`target\release\zed.exe`という実行ファイルが生成されるので、
これを実行すればZedが起動します。

<a id="#Chapter3"></a>

# 使ってみた感想
人並な感想ですが、1秒ぐらいで起動して軽くてサクサク動き、
実際に他のエディタとメモリ消費を比較してみると、確かにZedは軽量に見えます。
なお各種プラグインを無効化した上で比較していますが、実行環境によって諸条件が異なるのであくまで目安となります。

* Zed: 140MB
* VSCode: 304MB
* Cursor: 450MB
* Haystack: 347MB

欠点としてはVSCodeと比較した際のプラグインの少なさ、
またWindows限定ですが`WindowsからWSLに接続できない` and `WSL上で直接起動できない？`点です。
これはWSLユーザーとしては中々辛いので、今後のアップデートに期待していきたいです。

* [Cannot start on WSL 2](https://github.com/zed-industries/zed/discussions/14186)
