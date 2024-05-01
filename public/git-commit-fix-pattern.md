---
title: 過去のcommitを修正するいくつかの方法について
tags:
  - Git
  - GitHub
private: false
updated_at: '2024-05-01T17:16:04+09:00'
id: 63d1914978e4e15536b5
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- 発端や概要を記載 -->
# はじめに
この記事を読んでいるということは、止むに止まれぬ理由で過去のcommitを修正したいと考えているのでしょう。
私も様々な理由で毎週のように修正しているので、その気持ちはよくわかります...。

ただ修正するにしても方法がいくつかあり、どれを選ぶべきか迷ってしまいませんか？
そこでこの記事では過去のcommitを修正する方法と、使用するのに適した場面を合わせて解説します。

:::note warn
原則として過去のcommitの修正は、
`リモートリポジトリにプッシュする前のcommit`や、`メインブランチにマージ前のcommit`、
`個人的に使用しているリポジトリ`など他の作業者に影響が発生しないcommitに限定してください。
:::

<!-- 各チャプター -->
<a id="#Chapter1"></a>

# 1.commitに記録される内容について
そもそもcommitに何が記録されているか、そしてgit logでどのような情報が出力されるのか把握していますか？
これは決して馬鹿にしているわけではなく、標準的なオプションなしのgit logでは表示されない項目も存在するため見落としがちなポイントです。

まずcommitには以下の項目が含まれています。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/b552027e-4f1c-0caa-1bdd-9b89f51a5813.png)

| 項番 | 項目                | 内容                                                |
|:-----|:--------------------|:---------------------------------------------------|
| ①    |	Commit Hash        | commitを一意に識別するための SHA-1 ハッシュ値        |
| ②    |	Author             | commitを作成したユーザー名/メールアドレス            |
| ③    |	AuthorDate         | commitが作成された日時                              |
| ④    |	Commit (Commiter)  | commitをリポジトリに適用したユーザー名/メールアドレス |
| ⑤    |	CommitDate         | commitをリポジトリに適用した日時                     |
| ⑥    |	Commit Message     | commit時に作成したメッセージ                         |
| ⑦    |	Changes            | commitで行われた変更内容                             |

上記でcommit後に修正する可能性がある項目は、
`Author`、`AuthorDate`、`Commit`、`CommitDate`、`Commit Message`のいずれかかと思います。

また`Commit`と`CommitDate`をgit logで出力する場合、`--pretty`オプションで`fuller`を指定するか、
`format`を用いてカスタムフォーマットを設定しない限り出力されません。
なおオプションを指定しない場合のgit logは`--pretty=medium`オプションと同等です。

* `git log` 出力例
```
commit b00a1c8d57ad878d156fca2989726a30d5059c05 (HEAD)
Author: example-user <email@example.com>
Date:   Mon Jan 1 12:00:00 2024 +0900

    :wrench: express upgrade
```

* `git log --pretty=fuller` 出力例
```
commit b00a1c8d57ad878d156fca2989726a30d5059c05 (HEAD)
Author:     example-user <email@example.com>
AuthorDate: Mon Jan 1 12:00:00 2024 +0900
Commit:     example-user <example@example.com>
CommitDate: Mon Jan 1 12:00:00 2024 +0900

    :wrench: express upgrade
```

これらの点を理解していないと部分的な修正漏れを引き起こす可能性があるため、
上記の項目が存在することを前提に修正を行ってください。

なおgit logの仕様についてさらに深堀したい方は、以下公式ドキュメントをご参照ください。

* [2.3 Git の基本 - コミット履歴の閲覧](https://git-scm.com/book/ja/v2/Git-%E3%81%AE%E5%9F%BA%E6%9C%AC-%E3%82%B3%E3%83%9F%E3%83%83%E3%83%88%E5%B1%A5%E6%AD%B4%E3%81%AE%E9%96%B2%E8%A6%A7)


<a id="#Chapter2"></a>

# 2.過去commitの修正方法について
commitの修正は大きく分けて、`「単一のcommitを修正したい場合」`と`「複数のcommitを一括で修正したい場合」`のどちらかと思いますので、
それぞれに適した方法を解説します。

## (1) 単一のcommitを修正したい場合
`直前のcommit`や`過去の単一のcommit`を修正したい場合が対象です。
ちょっとした修正でよく利用するので、普段使いはこちらかと思います。

### ① git commit --amend
直前のcommitを修正したい場合はとりあえずこのコマンドです。
ただし過去のcommitは直接変更できないため、後述の`git rebase`と合わせて利用する必要があります。

また`git commit --amend`のみ実行すると`Commit Message`の編集と、`CommitDate`が修正時の時刻に更新されるだけなので、
その他項目も修正したい場合はオプションや環境変数で指定が必要になります。

#### Author、AuthorDateを変更したい場合
`Author`は`--author`オプション、
`AuthorDate`は`--date`オプションで値を指定することで変更できます。
```
git commit --amend \
  --author="example-user <email@example.com>" \
  --date="2024-01-01 12:00:00"
```

#### Commit、CommitDateを変更したい場合
`Commit`と`CommitDate`はオプションで指定できないため、
git config(~/.gitconfig)で設定するか、環境変数で設定する必要があります。

永続的に変更する場合は、`git config`で設定を更新してから`git commit --amend`を実行してください。
```
git config --global user.name "new-user"
git config --global user.email "new-email@example.com"
git commit --amend
```

修正対象のcommitのみ変えたい場合は、`git commit --amend`の実行時に環境変数として設定する必要があります。
また`CommitDate`を特定の時間を指定したい場合も、同じく環境変数で設定が必要です。
```
GIT_COMMITTER_DATE="2024-01-01 12:00:00" \
GIT_COMMITTER_NAME="example-user" \
GIT_COMMITTER_EMAIL="<email@example.com>" \
git commit --amend
```

その他の環境変数は割愛しますが、詳しく知りたい方は以下ドキュメントでご確認ください。
* [10.8 Gitの内側 - 環境変数](https://git-scm.com/book/ja/v2/Git%E3%81%AE%E5%86%85%E5%81%B4-%E7%92%B0%E5%A2%83%E5%A4%89%E6%95%B0)

### ② git rebase
一つ以上前の過去のcommitを修正したい場合はこちらを利用します。
`git commit --amend`と組み合わせて使う必要があり、以下の流れで修正できます。

#### 1.`git log --oneline`などで対象のcommit hashを確認
```
$ git log --oneline
b00a1c8 (HEAD) :wrench: express upgrade
fce919d :construction: add test page
b1d8db1 :construction: add react
4389323 :tada: first commit
```

#### 2.`git rebase -i`で修正対象の一つ前のcommit hashを指定
```
git rebase -i 4389323
```

#### 3.リベースモードに入るので、修正対象のcommitの`pick`を`edit`に書き換えて保存
```
edit b1d8db1 :construction: add react
pick fce919d :construction: add test page
pick b00a1c8 (HEAD) :wrench: express upgrade
```

#### 4.`git commit --amend`で修正
  ※各項目の修正方法は前述のgit commit --amendを参照

```
git commit --amend
```

#### 5.修正が完了したら`git rebase --continue`でリベースモードを終了
```
git rebase --continue
```


## (2) 複数のcommitを修正したい場合
`過去の複数のcommit`、または`過去のすべてのcommit`を修正したい場合が対象です。
影響範囲が大きいため、`git clone`したものに対して修正を実施することを推奨します。

### ① git filter-branch
gitに標準で含まれているため、追加のパッケージインストールや設定が不要なことが利点です。
ただし実行速度、安全性、使いやすさなどの観点で後述の`git filter-repo`の利用が推奨されているため、
基本的にはそちらをご利用ください。

* [7.6 Git のさまざまなツール - 歴史の書き換え](https://git-scm.com/book/ja/v2/Git-%E3%81%AE%E3%81%95%E3%81%BE%E3%81%96%E3%81%BE%E3%81%AA%E3%83%84%E3%83%BC%E3%83%AB-%E6%AD%B4%E5%8F%B2%E3%81%AE%E6%9B%B8%E3%81%8D%E6%8F%9B%E3%81%88)

#### 過去の複数のcommitを修正する場合

特定のcommitを複数指定して修正することはできないため、
条件式にてマッチしたものを置き換えるという方法になります。

例として以下であれば`GIT_COMMITTER_EMAIL`、または`GIT_AUTHOR_EMAIL`が`OLD_EMAIL`に合致する場合は、
`NEW_NAME`、`NEW_EMAIL`に置き換えられます。
```
git filter-branch --env-filter '
OLD_EMAIL="old-email@example.com"
NEW_NAME="new-user"
NEW_EMAIL="new-email@example.com"
if [ "$GIT_COMMITTER_EMAIL" = "$OLD_EMAIL" ]
then
    export GIT_COMMITTER_NAME="$NEW_NAME"
    export GIT_COMMITTER_EMAIL="$NEW_EMAIL"
fi
if [ "$GIT_AUTHOR_EMAIL" = "$OLD_EMAIL" ]
then
    export GIT_AUTHOR_NAME="$NEW_NAME"
    export GIT_AUTHOR_EMAIL="$NEW_EMAIL"
fi
' -- --all
```

#### 過去のすべてのcommitを修正する場合
すべてのcommitを修正する場合は、
以下の通り環境変数に値を指定するだけでOKです。

以下の場合は過去のcommitすべての`GIT_AUTHOR_NAME`、`GIT_AUTHOR_EMAIL`、`GIT_COMMITTER_NAME`、`GIT_COMMITTER_EMAIL`の値を置き換えています。
```
git filter-branch -f --env-filter \
  "GIT_AUTHOR_NAME='new-user'; \
   GIT_AUTHOR_EMAIL='new-email@example.com'; \
   GIT_COMMITTER_NAME='new-user'; \
   GIT_COMMITTER_EMAIL='new-email@example.com';"
```


### ② git filter-repo
git-filter-branchでも公式として`git filter-repo`の利用を推奨しており、
基本的にはこちらを利用します。

* [git-filter-branch - Rewrite branches](https://git-scm.com/docs/git-filter-branch)
* [git-filter-repo](https://github.com/newren/git-filter-repo/)

利用にあたりインストールする必要があるので、以下の方法で導入してください。
```text:Linux (RedHat系)
sudo yum install git-filter-repo
```
```text:Linux (Debian系)
sudo apt install git-filter-repo
```
```text:Mac
brew install git-filter-repo
```
```text:Windows (pip)
pip install git-filter-repo
```

#### 過去の複数のcommitを修正する場合

特定のcommitを複数指定して修正することはできないため、
条件式にてマッチしたものを置き換えるという方法になります。

例として以下であれば`commit.committer_name`、または`commit.author_name`が`old_name`に合致する場合は、
`new_name`、`new_email`に置き換えられます。
```
git filter-repo --commit-callback '
old_name = b"old-user"
new_name = b"new-user"
new_email = b"new-email@example.com"

if commit.committer_name == old_name :
  commit.committer_name = new_name
  commit.committer_email = new_email
if commit.author_name == old_name :
  commit.author_name = new_name
  commit.author_email = new_email
'
```

またauthorとcommitterの情報が同じ場合は、
コールバックのオプションを利用することでより簡潔なコマンドで修正できます。
* [git-filter-repo(1) Manual Page CALLBACKS](https://htmlpreview.github.io/?https://github.com/newren/git-filter-repo/blob/docs/html/git-filter-repo.html#CALLBACKS)
```
git-filter-repo \
  --name-callback 'return name.replace(b"old-user", b"new-user")' \
  --email-callback 'return email.replace(b"old-email@email.com", b"new@email.com")'
```

#### 過去のすべてのcommitを修正する場合
すべてのcommitを修正する場合は、
以下の通り変数に値を指定するだけでOKです。
```
git filter-repo --commit-callback '
new_name = b"new-user"
new_email = b"new-email@example.com"

commit.committer_name = new_name
commit.committer_email = new_email
commit.author_name = new_name
commit.author_email = new_email
'
```

またauthorとcommitterの情報が同じ場合は、
コールバックのオプションを利用することでより簡潔なコマンドで修正できます。
```
git-filter-repo \
  --name-callback 'return b"new-user"' \
  --email-callback 'return b"new-email@email.com"'
```

<a id="#Chapter3"></a>

# 3.最後に
以上、過去のcommitの修正方法についてでした。

しつこいようですが、
個人のリポジトリでない限り他の人とのコンフリクトが発生する可能性が高いため、実際に使用する際は細心の注意を払ってください。
ただし、いざというときに役立つ内容だと思いますので必要な時にはぜひご活用ください。

~~まあ、そもそも後から修正したくなるようなコミットをしないことが最善ではありますが...。~~

<a id="#Chapter4"></a>

## 参考ドキュメント
本記事の作成に際して、以下のドキュメントを参照させていただきました。
この場を借りてお礼申し上げます。

* [2.3 Git の基本 - コミット履歴の閲覧](https://git-scm.com/book/ja/v2/Git-%E3%81%AE%E5%9F%BA%E6%9C%AC-%E3%82%B3%E3%83%9F%E3%83%83%E3%83%88%E5%B1%A5%E6%AD%B4%E3%81%AE%E9%96%B2%E8%A6%A7#rpretty_format)
* [7.6 Git のさまざまなツール - 歴史の書き換え](https://git-scm.com/book/ja/v2/Git-%E3%81%AE%E3%81%95%E3%81%BE%E3%81%96%E3%81%BE%E3%81%AA%E3%83%84%E3%83%BC%E3%83%AB-%E6%AD%B4%E5%8F%B2%E3%81%AE%E6%9B%B8%E3%81%8D%E6%8F%9B%E3%81%88)
* [10.8 Gitの内側 - 環境変数](https://git-scm.com/book/ja/v2/Git%E3%81%AE%E5%86%85%E5%81%B4-%E7%92%B0%E5%A2%83%E5%A4%89%E6%95%B0)
* [GitのCommitユーザを修正する方法](https://qiita.com/y10exxx/items/dcea0e39788d649ca8ba)
* [【git filter-repo】リポジトリーのファイルとコミット履歴をディレクトリ階層を変えつつ別のリポジトリーに移動する](https://qiita.com/shimamura_io/items/5f0dd5346dd22edc06ad#3-git-filter-repo-%E3%81%AE%E5%AE%9F%E8%A1%8C)
* [git logのオプションと綺麗にツリー表示するためのエイリアス](https://qiita.com/kawasaki_dev/items/41afaafe477b877b5b73#--pretty)
