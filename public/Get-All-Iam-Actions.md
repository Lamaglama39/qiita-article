---
title: 【AWS】IAMポリシーで設定できるアクションがいくつあるか、あなたは知っていますか？
tags:
  - 'AWS'
  - 'IAM'
  - 'Python'
  - 'スクレイピング'
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- 発端や概要を記載 -->
# はじめに
皆さんはIAMポリシーで設定できるアクションがいくつあるかご存じですか？
私は知らないのですが、[例のドキュメント](https://docs.aws.amazon.com/service-authorization/latest/reference/reference_policies_actions-resources-contextkeys.html)を見ながらIAMポリシーの権限設定を検証しているときふと、一体いくつあるのかなぁと気になりました。

という訳で今回はIAMポリシーで設定できるアクションの総数を調べてみました。

* [例のドキュメント](https://docs.aws.amazon.com/service-authorization/latest/reference/reference_policies_actions-resources-contextkeys.html)

:::note warn
本記事の調査結果は2024/3/7時点のものです。
今後AWSのアップデートによって変動する点についてはご留意ください。
:::


<!-- 各チャプター -->
<a id="#Chapter1"></a>

# 1.調査方法
まず思い当たるのは`AWS CLI`や`AWS SDK`などを利用する方法ですが、
残念ながら現時点(2024年3月)ではアクションの一覧を表示する手段は用意されていません。
`aws iam list-actions`的なコマンドがあって一覧を出力できたらとても楽だったのですが...。

なので[例のドキュメント](https://docs.aws.amazon.com/service-authorization/latest/reference/reference_policies_actions-resources-contextkeys.html)から地道にスクレイピングすることにしました。
解説するポイントはそんなにないですが、
(1)で取得したサービスのURLごとに(2)～(3)の処理を繰り返しています。
また実行時間が長いので(1700秒ぐらい)、一時的なネットワークエラー発生時に備えリトライ処理を入れています。
```text:実行内容
(1).トップページにてすべてサービスのURLを取得
(2).サービスのページにてActionsの一覧を取得
(3).Actionsの一覧をファイル(actions.txt)へ出力
```

```python:get-all-iam-action.py
import requests
from bs4 import BeautifulSoup
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry
import time

def requests_retry_session(retries=3, backoff_factor=0.3, status_forcelist=(500, 502, 504), session=None):
    """一時的なネットワークエラー向けのリトライ処理"""
    session = session or requests.Session()
    retry = Retry(
        total=retries,
        read=retries,
        connect=retries,
        backoff_factor=backoff_factor,
        status_forcelist=status_forcelist,
    )
    adapter = HTTPAdapter(max_retries=retry)
    session.mount('http://', adapter)
    session.mount('https://', adapter)
    return session

def fetch_html(url):
    """指定されたURLからHTMLを取得して解析した結果を返す"""
    session = requests_retry_session()
    response = session.get(url)
    response.encoding = response.apparent_encoding
    return BeautifulSoup(response.text, 'html.parser')

def fetch_service_actions(base_url, service_url):
    """サービスURLからアクション名を抽出してリストで返す"""
    bs_obj = fetch_html(base_url + service_url)
    service_name = bs_obj.find('code', class_='code').text
    table = bs_obj.select_one('.table-container table')
    actions = []

    if table:
        rows = table.find_all('tr')[1:]
        for row in rows:
            first_td = row.find('td')
            if first_td:
                actions.append(first_td.get_text(strip=True))

    uppercase_actions = [action for action in actions if action and action[0].isupper()]
    cleaned_actions = [action.replace('[permission only]', '') for action in uppercase_actions]
    return [f'{service_name}:{action}\n' for action in cleaned_actions]

# 実行時間計測の開始
start_time = time.time()

# 基本URLと全サービス一覧のページURL
base_url = 'https://docs.aws.amazon.com/service-authorization/latest/reference/'
all_service_url = 'reference_policies_actions-resources-contextkeys.html'

# アクション一覧の出力先ファイル
output_file = 'actions.txt'

# 全サービスのURLを取得
bs_obj = fetch_html(base_url + all_service_url)
links = bs_obj.select('.highlights a')
urls = [link.get('href').lstrip('./') for link in links]

# 取得したURLごとにアクション一覧を出力
for relative_url in urls:
    fullname_actions = fetch_service_actions(base_url, relative_url)
    
    with open(output_file, 'a') as file:
        file.writelines(fullname_actions)
    
    # サーバーへの負荷軽減のために待機
    time.sleep(3)

# 実行時間計測の終了
end_time = time.time()

# 実行時間の計算と表示
elapsed_time = end_time - start_time
print(f'実行時間: {elapsed_time:.2f}秒')

```

出力されるアクション一覧のファイル(actions.txt)は以下の形式としており、
実際にIAMポリシーで設定するときの形式(`サービスプレフィックス`:`アクション名`)で出力しています。
```text:actions.txt 出力例
account:CloseAccount
account:DeleteAlternateContact
account:DisableRegion
account:EnableRegion
account:GetAccountInformation
```

また出力したアクション一覧のファイルから逆引きして、
サービスプレフィックスの一覧を別ファイル(service_prefix.txt)へを出力しました。
```python:count_service_prefix.py
# アクション一覧の入力元ファイル
# サービスプレフィックス一覧の出力先ファイル
input_file = 'actions.txt'
output_file = 'service_prefix.txt'

# アクション一覧ファイルから内容を読み込み、各行の":"より前を抽出
with open(input_file, 'r') as file:
    lines = file.readlines()

# 重複する値を排除してuniqなリストを作成
unique_prefixes = set(line.split(':')[0] for line in lines)

# uniqなリストをサービスプレフィックス一覧ファイルへ出力
with open(output_file, 'w') as output_file:
    for prefix in sorted(unique_prefixes):
        output_file.write(prefix + '\n')

```

<a id="#Chapter2"></a>

# 2.結果
結果は以下の通りで、なんとアクションの総数は驚異の`17,163`個でした...、
これだけの数の権限を適切に制御しているAWSはすごいなぁ、と改めて畏敬の念を抱きました。
~~というかいくら何でも流石に多すぎませんか...？~~

| 項目                     | 総数       | 説明                                                               |
|:------------------------|:-----------|:-------------------------------------------------------------------|
|  サービス数              | 405個      | トップページでリンクされているサービスのURLの総数                      |
|  サービスプレフィックス   | 386個      | アクション設定時に指定するプレフィックス (例 `ec2`:DescribeInstances) |
|  アクション数            | 17,163個   | 各サービスのアクションの総数                                         |

またサービス数とサービスプレフィックス数が異なる理由については、
一部のサービスでは同一のサービスプレフィックスが設定されているためです。
例えば`Amazon API Gateway Management`と`Amazon API Gateway Management V2`はそれぞれ別ページとして存在しますが、
サービスプレフィックスはどちらも`apigateway`です。

* [Amazon API Gateway Management](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonapigatewaymanagement.html)
* [Amazon API Gateway Management V2](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonapigatewaymanagementv2.html)

あくまで憶測ですが、サービスリリース後に追加された大きな変更や機能は別ページで管理しているけど、
実態は一つのサービスだからこのような形式になってのではと考えられます。


<a id="#Chapter3"></a>

# 3.最後に
以上、IAMポリシーで設定可能なアクションの総数に関するお話でした。

最後までお読みいただいたことに感謝しますが、お察しの通りこれは`無駄な知識`かもしれません...。
しかしあの某アンミカさんがおっしゃったように、
`「IAMポリシーで設定できるアクションって17,163個あんねん」`という風にネタにしていただけると嬉しいです。
