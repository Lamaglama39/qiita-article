---
title: QiitaのタグをQiita APIで取得して見えてきた傾向について
tags:
  - Qiita
  - Python
  - API
  - CLI
private: false
updated_at: '2025-01-27T15:30:42+09:00'
id: 24f06d205753b6b11dc5
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- 発端や概要を記載 -->
# はじめに
皆さんはQiita CLI使っていますか？
2023/8/9の正式リリースからまだ約一年半程度ではありますが、
`好きなエディターで記事が執筆できる`ことは勿論、`Githubでの管理がしやすい`という利点もあり、
執筆の全てをQiita CLIで行っているユーザーも多いのではないでしょうか。
(筆者もそんなユーザーの一人です)

* [自分のエディタで記事投稿ができる、Qiita CLIの正式版を公開しました！](https://blog.qiita.com/release-qiita-cli-general/)

ただ個人的にQiita CLIで困っている点があり、それは`どんなタグが存在しているか確認できない`ことです。
なので似たような記事を探してタグを確認したり、
以下の一覧から探したりしているのですが、正直少々手間です。

* [Qiita タグ一覧](https://qiita.com/tags)

ということで`Qiita API`を利用して2025/01/25時点のタグを取得して一覧化して、
その結果と見えてきた傾向について記事にしてみました。

:::note info
取得結果は以下リポジトリにJSON形式でまとめています。
なお前述の通り、本記事作成時点の情報のためその点はご留意ください。

* [qiita-all-tags](https://github.com/Lamaglama39/qiita-tags/blob/main/dist/tags.json)
:::



<!-- 各チャプター -->
<a id="#Chapter1"></a>

# 取得方法

## Qiita APIの仕様と規約について

仕様と規約について全文は以下に記載がありますが、一部抜粋すると、
* [Qiita API・スクレイピングについて](https://help.qiita.com/ja/articles/qiita-api)

まずQiita APIの利用において、
`アプリ等に広告を設置して収益化しない限り`は利用が許可されています。
```txt:Qiita APIを利用したアプリ等の作成について
Qiita APIでご用意している機能の範囲内でしたら、アプリ等を開発いただいて問題ございません。
ただし、アプリ等に広告を設置して収益化されることに関しては Qiita利用規約 違反となります。
```

逆に`スクレイピングについては許可していない`ため、
基本的には`Qiita APIを利用してデータを取得する`ことになります。

```txt:スクレイピングについて
スクレイピングはサーバーの負荷上昇への懸念があるため、Qiitaへのスクレイピングは許可しておりません。
```

また利用制限として、以下二点があります。
本記事では`認証している状態での利用を想定`して進めます。
* [Qiita API 利用制限](https://qiita.com/api/v2/docs#%E5%88%A9%E7%94%A8%E5%88%B6%E9%99%90)
  - (1).`認証している状態`では`ユーザーごと`に1時間に1000回までリクエスト可能
  - (2).`認証していない状態`では`IPアドレスごと`に1時間に60回までリクエスト可能


## Qiita API 利用方法について

### (1).アクセストークン発行
まずはアクセストークンを発行する必要があります。
Qiitaのアカウントがあれば誰でも取得でき、料金がかかるものでもないので気軽に発行して大丈夫です。

アクセストークンは以下で発行できます。
* [個人用アクセストークン](https://qiita.com/settings/applications)

付与する権限は利用するAPIに合わせますが、
今回は参照のみ利用するので`read_qiita`を付与します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/6022e678-b00d-2765-2c57-7229d4bf66b9.png)

### (2).APIリクエスト/レスポンス
あとはリクエスト時に、発行したアクセストークンを指定するだけです。

ここではcurlを利用したリクエストを例として挙げます、
以下は`指定したユーザーのプロフィール情報をレスポンスとして返却するAPI`を叩いています。
```bash:リクエスト例
curl -H 'Authorization: Bearer <アクセストークン>' 'https://qiita.com/api/v2/users/<ユーザー名>' | jq
```

正常に実行されれば以下のようなユーザー情報が出力されます。
(例としてQiita公式アカウントを指定しています)
```json:レスポンス例
{
  "description": "Qiita公式アカウントです。Qiitaに関するお問い合わせに反応したり、お知らせなどを発信しています。",
  "facebook_id": "qiita",
  "followees_count": 2,
  "followers_count": 624174,
  "github_login_name": "qiitan",
  "id": "Qiita",
  "items_count": 46,
  "linkedin_id": "",
  "location": "Qiitaの中",
  "name": "Qiita キータ",
  "organization": "Qiita",
  "permanent_id": 88,
  "profile_image_url": "https://s3-ap-northeast-1.amazonaws.com/qiita-image-store/0/88/ccf90b557a406157dbb9d2d7e543dae384dbb561/large.png?1575443439",
  "team_only": false,
  "twitter_screen_name": "Qiita",
  "website_url": "https://qiita.com"
}
```

## タグの取得方法について
タグを取得するには、`/api/v2/tags`を利用します。

* [GET /api/v2/tags](https://qiita.com/api/v2/docs#get-apiv2tags)

また以下のクエリパラメータを利用します。
なお仕様上100ページ以降は取得できないので、全タグの取得は難しいそうです。
(もちろんスクレイピングすれば全て取得できますが、規約上禁止されているのでやめてください...)

* page - ページ番号 (1から100まで)
* per_page - 1ページあたりに含まれる要素数 (1から100まで)
* sort - 並び順 (countで記事数順、nameで名前順)

一例としてcurlを利用した場合、以下のように取得できます。
以下は`記事数順で1ページ目の要素を3個`取得しています。
```bash:リクエスト例
curl -H 'Authorization: Bearer <アクセストークン>' 'https://qiita.com/api/v2/tags?page=1&per_page=3&sort=count' | jq
```

```json:レスポンス例
[
  {
    "followers_count": 208673,
    "icon_url": "https://s3-ap-northeast-1.amazonaws.com/qiita-tag-image/0ee2c162b0573701a6baf468f4d30549f8d03e9b/medium.jpg?1660803670",
    "id": "Python",
    "items_count": 88248
  },
  {
    "followers_count": 179078,
    "icon_url": "https://s3-ap-northeast-1.amazonaws.com/qiita-tag-image/12ab79a9d2e703932a2c08dc6a4bcc9fb544f5c3/medium.jpg?1650353657",
    "id": "JavaScript",
    "items_count": 59602
  },
  {
    "followers_count": 79616,
    "icon_url": "https://s3-ap-northeast-1.amazonaws.com/qiita-tag-image/e92cc40a9770111ffa5833b87b3fb7e04a0a2b5e/medium.jpg?1650353581",
    "id": "AWS",
    "items_count": 46756
  }
]
```

今回は以下のPythonスクリプトで取得してみます、
アクセストークンは環境変数(QIITA_TOKEN)に埋め込んでいます。

<details><summary>サンプルコード</summary>

```python:main.py
import os
from dotenv import load_dotenv
import time


def get_tag(token, page, per_page):
    import requests

    base_url = "https://qiita.com/api/v2/tags"
    headers = {"Authorization": f"Bearer {token}"}

    url = f"{base_url}?page={page}&per_page={per_page}&sort=count"
    response = requests.get(url, headers=headers)

    if not response.ok:
        return {
            "error": True,
            "status_code": response.status_code,
            "reason": response.reason,
            "message": response.text
        }
    else:
        return {
            "error": False,
            "status_code": response.status_code,
            "reason": response.reason,
            "message": response.json()
        }


def write_file(data, filename="tags.json"):
    import json

    if os.path.exists(filename):
        with open(filename, "r", encoding="utf-8") as f:
            try:
                existing_data = json.load(f)
            except json.JSONDecodeError:
                existing_data = []
    else:
        existing_data = []

    if isinstance(existing_data, list):
        if isinstance(data, list):
            existing_data.extend(data)
        else:
            existing_data.append(data)
    else:
        raise ValueError("Format of the existing JSON file was unexpected")

    with open(filename, "w", encoding="utf-8") as f:
        json.dump(existing_data, f, ensure_ascii=False, indent=2)


def main():
    load_dotenv()

    token = os.environ.get("QIITA_TOKEN")
    page = 1
    per_page = 100

    while True:
        if page > 100:
            print("Get all data")
            break

        data = get_tag(token, page, per_page)
        if data["error"]:
            print(f"Failed to get data: page {page}")
            print(data)
            break

        write_file(data["message"])
        print(f"Get data: page {page}")

        page += 1
        time.sleep(2)


if __name__ == "__main__":
    main()
```

</details>

<a id="#Chapter2"></a>

# 取得結果

Pythonスクリプトを回すこと約3分、
100ページ分取得できたので、この時点で`10,000個タグが存在する`ということがわかりました、
既に多すぎる気がするのですがどうなんでしょうか...。
* [qiita-all-tags](https://github.com/Lamaglama39/qiita-tags/blob/main/dist/tags.json)

<a id="#Chapter3"></a>

# Qiitaにおけるタグの傾向と考察
前述のQiita APIの結果と合わせ、
100ページ以降を手動でちまちま確認したのでその結果から考察します。

まずタグのページ数ですが、何と現状で`1014ページ`まで存在していました！
なので単純計算で `1014ページ × 100個 = 101,400個`のタグが存在していることになります。
* [タグ一覧 1014ページ目](https://qiita.com/tags?page=1014)

以下記事では2011年9月にサービス開始とのことなので、約14年の間でこれだけの数のタグが生まれたことになります。
* [日本最大級のエンジニアコミュニティ「Qiita」、会員数100万人突破！突破を記念し、2つのキャンペーンを実施](https://corp.qiita.com/releases/2023/07/one-million/)


そしてなんと`502ページ目から1記事にしか使われていないタグ`が含まれていました。
* [タグ一覧 502ページ目](https://qiita.com/tags?page=502)

はい、勘の良い方はお察しかもしれませんが、
これは`タグの約半数が1記事にしか使われていない`ということを意味しています...。

---

この1記事にしか使われていないタグ達について、
具体的なタグ名は記載しませんが以下のような共通点がありました。

* おそらく日本ではマニアックすぎる分野
* タイポなどの表記ゆれ
* オリジナルティーがあふれている独自の造語
* 技術とは関係のない一般的な慣用句

あまり個人がとやかく言うことではないのでしょうが、
`前提としてタグは記事の内容を端的に表している必要がある`と考えます。
なので、少なくとも誤字脱字によってオリジナルのタグを生成することは避けたいですね...。

なお、Qiita側としてもユーザー側としても認知負荷が上がるため、無尽蔵にタグが増加することは望ましくないはずです。
しかし、自由な技術記事の投稿サービスとして提供している以上、
Qiita側の独断で削除するわけにもいかないので、結果としてこれだけの数になったのではないでしょうか...。

<a id="#Chapter4"></a>

# さいごに
以上、普段使ってはいるもののあまり意識する機会の少ない、`Qiitaのタグ`に関するお話でした。
これを機に、改めて自身が利用しているタグが適切かどうか、振り返るきっかけになれば幸いです。

もちろんタグは個人が自由に設定できるので、完全オリジナルなものを使うこともできます。
ただ、せっかく皆さんが`愛を込めて書いた記事`ですから、より多くの人に読まれるように適切なタグを活用していきたいですね。
