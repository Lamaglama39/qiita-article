---
title: わりと使うけど忘れがちなGitコマンド集
tags:
  - Git
  - GitHub
  - cmd
private: false
updated_at: '2024-11-20T00:42:13+09:00'
id: c534891ed24dc1dadf46
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- 発端や概要を記載 -->
# はじめに
備忘としてわりと使うけど忘れがちなGitコマンドをまとめました。
(随時更新予定)


### リモートリポジトリ操作

* リモートリポジトリの参照
```
git remote -v
```

* リモートリポジトリを新規追加
```
git remote add origin <url>
```

* リモートリポジトリを変更
```
git remote set-url origin  <url>
```

* 削除されているリモートブランチを、ローカルブランチから削除
```
git fetch --prune
```

* リモートブランチの削除
```
git push --delete origin <branch>
```

* リモートブランチの一覧を表示
```
git ls-remote
```


### ローカルブランチ操作

* ローカルブランチ名を変更
```
git branch -m <old-name> <new-name>
```

* ローカルブランチを削除
```
git branch -d <branch>
```

* マージ済みのブランチを一覧表示
```
git branch --merged
```

* 未マージのブランチを一覧表示
```
git branch --no-merged
```

* 新規ブランチを作成して移動
```
git checkout -b <new-branch>
```

* 削除したブランチの復元
```
git reflog
git branch <deleted-branch> HEAD@{n}  # ブランチの復元
git checkout <deleted-branch>
```

* 他ブランチの差分を取り込む
```
git rebase <branch>
```


### コミット操作

* ステージングに追加
```
git add <file>
```

* コミットを作成
```
git commit -m "message"
```

* ステージングした変更を直前のコミットに追加
```
git commit --amend --no-edit
```

* 直前のコミットメッセージを修正
```
git commit --amend -m "new message"
```

* 過去のコミットを編集
```
git rebase -i HEAD~<n>
```

* 特定のコミットをブランチに取り込む
```
git cherry-pick <commit>
```

* 削除したコミットの復元
```
git log -- <filename>
git restore --source=commitID^ -- <filename>
git add <filename>
git commit -m "Restore filename"
```


### 履歴のリセット

* 直前の git add をすべて取り消し
```
git reset --mixed HEAD
```

* 直前のコミットを取り消し、変更は保持
```
git reset --soft HEAD^
```

* 直前のコミット・変更内容をすべて取り消し
```
git reset --hard HEAD^
```

* 直前のリセット操作を取り消し
```
git reset --hard ORIG_HEAD
```

* 一時的に変更を退避
```
git stash            # 変更内容を一時退避
git stash show       # スタッシュの内容を確認
git stash pop        # スタッシュから変更を復元
```

### 差分確認

* 変更のみを確認
```
git diff [--diff-filter=M]
```

* 直前のコミットの差分を確認
```
git diff HEAD^ HEAD
```

* 差分のあるファイル一覧を表示
```
git diff --name-only <commit>
```

* Git管理外のファイルを比較
```
git diff --no-index -- <path>
```

### ログ管理

* コミットメッセージを一行で表示
```
git log --oneline
```

* 差分付きのログを表示
```
git log -p
```

* ファイルごとの変更量を表示
```
git log --stat
```

* コミットログツリーを可視化
```
git log --graph
```

* 特定のコミットの詳細を表示
```
git show <commit>
```

* 各行の変更者を表示
```
git blame <file>
```

* ファイルの変更履歴を表示
```
git log -- <file>
```

### サブモジュール操作

* サブモジュール追加
```
git submodule add <url>
```

* サブモジュール初期化
```
git submodule init
```

* サブモジュール初期化 (git clone時に初期化)
```
git clone --recursive <url>
```

* サブモジュールも含めて一括で初期化 & 更新
```
git submodule update --init --recursive --remote
```

### オブジェクト整合性と不要データ削除

* オブジェクトの整合性をチェック
```
git fsck
```

* 差分の圧縮と不要オブジェクト削除
```
git gc --aggressive --prune=all
```

### その他

* Git追跡対象外ファイルを削除
```
git clean -f
```

* 指定バージョンのアーカイブを作成
```
git archive -o <file>.zip HEAD
```
