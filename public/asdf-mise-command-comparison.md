---
title: asdf/mise コマンド比較
tags:
  - 'asdf'
  - 'mise'
  - 'cmd'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- 発端や概要を記載 -->
# はじめに
私事ですが、最近になってバージョン管理をasdfからmiseへ移行しました。
基本的にasdfと互換性のあるmiseですが一部形式が異なるコマンドがあったため、備忘として本記事を書きました。

なお`あのコマンドもよく使うよ～`とか、
`このコマンドの比較もほしいな～`などご意見があればお気軽にコメントください。

<!-- 各チャプター -->
<a id="#Chapter1"></a>

# コマンドまとめ
コマンド例の`terraform`はお好きなプラグイン名、`1.8.5`はお好きなプラグインバージョンに置き換えてください。

## プラグイン参照系

| 使用用途                                            | asdf                       | mise                      |
|:---------------------------------------------------|:---------------------------|:--------------------------|
| コマンドのリファレンスを表示                          |	asdf                       | mise	                     |
| インストール可能な全てのプラグインを表示               | asdf plugin list all       | mise plugins ls-remote    |
| インストール可能な全てのバージョンを表示               | asdf list all terraform    | mise ls-remote terraform  |
| プラグインの最新バージョンを表示                       | asdf latest terraform      | mise latest terraform     |
| 利用しているプラグインのバージョンを表示                | asdf current               | mise current              |
| インストール済みのプラグインのバージョンを表示           | asdf list                  | mise list                 |

## プラグイン管理系

| 使用用途                                                | asdf                                 | mise                                |
|:-------------------------------------------------------|:-------------------------------------|:------------------------------------|
| プラグインの追加                                         | asdf plugin add terraform            | mise plugin add terraform           |
| プラグインの特定バージョンのインストール                   | asdf install terraform 1.8.5         | mise install terraform@1.8.5        |
| 使用するプラグインのバージョン固定 (local)                 | asdf local terraform 1.8.5           | mise local terraform@1.8.5          |
| 使用するプラグインのバージョン固定 (global)                | asdf global terraform 1.8.5          | mise global terraform@1.8.5         |
| プラグインの追加～バージョン固定まで一括で実行              | -                                    | mise use --global terraform@1.8.5   |
| プラグインをアップデート                                  | asdf plugin update terraform         | mise plugins update terraform       |
| 全てのプラグインをアップデート                            | asdf plugin update --all             | mise plugins update                 |
| プラグインのアンインストール                              | asdf plugin remove terraform         | mise plugins uninstall terraform    |
| プラグインの特定バージョンのアンインストール                | asdf uninstall terraform 1.8.5       | mise uninstall terraform@1.8.5      |
| 利用していないバージョンをまとめてアンインストール          | -                                     | mise prune terraform                |

## asdf/mise 自体の管理系

| 使用用途                                                | asdf                                 | mise                                |
|:-------------------------------------------------------|:-------------------------------------|:------------------------------------|
| asdf/mise バージョンアップ                               | asdf update                          | mise self-update                    |
| asdf/mise アンインストール                               | -                                    | mise implode                        |

<a id="#Chapter2"></a>

## 参考ドキュメント

* [asdf コマンドリファレンス](https://asdf-vm.com/manage/commands.html)
* [mise コマンドリファレンス](https://mise.jdx.dev/cli/)
