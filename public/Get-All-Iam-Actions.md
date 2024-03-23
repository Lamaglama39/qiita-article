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
本記事の調査結果は2024/3/22時点のものです。
今後AWSのアップデートによって変動する点についてはご留意ください。
:::


<!-- 各チャプター -->
<a id="#Chapter1"></a>

# 1.調査方法

### (1)AWS CLIとかAWS SDKは...？
まず思い当たるのは`AWS CLI`や`AWS SDK`などを利用する方法ですが、
残念ながら現時点(2024年3月)ではアクションの一覧を表示する手段は用意されていません。
`aws iam list-policy-actions`的なコマンドがあって一覧を出力できたらとても楽だったのですが...。

### (2)マネジメントコンソールは...？
次にマネジメントコンソールはどうでしょうか。
以下のように明らかに過剰な権限を付与したIMAポリシーを用意してみますが...、
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/cf726032-8460-12eb-67da-a8294ee41469.png)

残念ながらサービス数しか確認ができません。
というかこの時点で405個サービスがあって正直驚きました。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/66c7557c-e322-b60b-4219-d05273c1521f.png)

なお個別にサービスの詳細を見ることでアクションを数を確認できますが、
流石にこれを405回繰り返すのはつらい...。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3491064/1f4b148f-796d-5e5e-dbfd-204f91ed45c0.png)

### ③もうドキュメントから引き抜くしか...。
というわけで[例のドキュメント](https://docs.aws.amazon.com/service-authorization/latest/reference/reference_policies_actions-resources-contextkeys.html)から地道にスクレイピングすることにしました。

処理内容は以下の通りで、
(1)で取得した各サービスのURLごとに(2)～(3)の処理を繰り返しています。
```
(1).トップページにてすべてサービスのURLを取得
(2).各サービスのページにてサービスプレフィックス、アクションの一覧を取得
(3).取得した情報をファイルへ出力
```

次にアクションの抽出条件についてですが、大抵はアクションごとにドキュメントが用意されており`aタグ`でリンクとなっているため、
hrefに以下を含むものを条件としました。
 * `https://docs.aws.amazon.com/`
 * `https://aws.amazon.com/`
 * `${APIReferenceDocPage}`
 * 末尾に`.html`を含むもの

本来ならばテーブル要素の「Actions」列のデータだけを抽出したかったのですが、
サービスによってテーブルの構造が異なるため、仕方なく広範囲のデータを取得し後からフィルタリングすることにしました...。

なお以下のサービスのドキュメントは上記条件に該当せず取得できなかったため、
個別にカウントして出力結果に合計`34`加算しています。
 * [AWS DeepLens](https://docs.aws.amazon.com/service-authorization/latest/reference/list_awsdeeplens.html)
 * [Amazon Elastic Inference](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonelasticinference.html)
 * [AWS Marketplace Commerce Analytics Service](https://docs.aws.amazon.com/service-authorization/latest/reference/list_awsmarketplacecommerceanalyticsservice.html)
 * [AWS Marketplace Entitlement Service](https://docs.aws.amazon.com/service-authorization/latest/reference/list_awsmarketplaceentitlementservice.html)

```python:get-all-iam-action.py
import requests
from bs4 import BeautifulSoup
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry
import time
import csv
import os
import re

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

    html_pattern = re.compile(r'.*\.html')

    actions = table.find_all('a', href=lambda href: href and (
        "https://docs.aws.amazon.com/" in href or
        "https://aws.amazon.com/" in href or
        "${APIReferenceDocPage}" in href or
        html_pattern.search(href)
    ))

    format_actions = [
    f'{service_name}:{action.text.replace("[permission only]", "").strip()}\n'
    for action in actions
    ]
    unique_formatted_actions = list(dict.fromkeys(format_actions))

    service_info = [base_url + service_url,service_name,str(len(unique_formatted_actions))]
    return unique_formatted_actions,service_info


# 実行時間計測 開始
start_time = time.time()

# URL 出力先ファイル
base_url = 'https://docs.aws.amazon.com/service-authorization/latest/reference/'
all_service_url = 'reference_policies_actions-resources-contextkeys.html'
actions_file = 'iam_actions.txt'
csv_file_path = 'iam_service.csv'

# 全サービス URL取得
bs_obj = fetch_html(base_url + all_service_url)
links = bs_obj.select('.highlights a')
urls = [link.get('href').lstrip('./') for link in links]

# URLごとにアクション一覧を取得
for relative_url in urls:
    action_list, service_list = fetch_service_actions(base_url, relative_url)

    # アクション一覧 書き込み
    with open(actions_file, 'a') as file:
        file.writelines(action_list)

    # サービス一覧 書き込み
    file_exists = os.path.isfile(csv_file_path)
    with open(csv_file_path, 'a', newline='') as csvfile:
        writer = csv.writer(csvfile)
        if not file_exists:
            writer.writerow(['URL', 'Service Name', 'Number of Actions'])
        writer.writerow(service_list)

    time.sleep(2)

# 実行時間の計算と表示
end_time = time.time()
elapsed_time = end_time - start_time
print(f'実行時間: {elapsed_time:.2f}秒')

```

出力されるアクション一覧のファイル(iam_actions.txt)は以下の通り、
実際にIAMポリシーで設定するときの形式(`サービスプレフィックス`:`アクション名`)で出力しています。
```text:iam_actions.txt 出力例
account:CloseAccount
account:DeleteAlternateContact
account:DisableRegion
```

またサービス一覧のファイル（iam_service.csv）は、以下の通りCSV形式で出力しています
```text:iam_service.csv 出力例
URL,Service Name,Number of Actions
https://docs.aws.amazon.com/service-authorization/latest/reference/list_awsaccountmanagement.html,account,13
https://docs.aws.amazon.com/service-authorization/latest/reference/list_awsactivate.html,activate,8
https://docs.aws.amazon.com/service-authorization/latest/reference/list_alexaforbusiness.html,a4b,96
```

<a id="#Chapter2"></a>

# 2.結果
結果は以下の通りで、アクションの総数は驚異の`16,386`でした、
これだけの数の権限を適切に制御しているAWSはすごいなぁ、と改めて畏敬の念を抱きました。

| 項目                     | 総数      | 概要                                                               |
|:------------------------|:----------|:-------------------------------------------------------------------|
|  サービス                | 405      | トップページでリンクされているサービスのURL                            |
|  サービスプレフィックス   | 386      | アクション設定時に指定するプレフィックス (例 `ec2`:DescribeInstances)  |
|  アクション              | 16,386   | 各サービスのアクション (例 ec2:`DescribeInstances`)                  |

`サービス`と`サービスプレフィックス`の数に相違があるのは、
一部のサービスでは同じサービスプレフィックスが使用されているためです。
たとえば、`Amazon API Gateway Management`と`Amazon API Gateway Management V2`は個別のドキュメントで管理されていますが、
サービスプレフィックスは両方とも`apigateway`となっています。

* [Amazon API Gateway Management](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonapigatewaymanagement.html)
* [Amazon API Gateway Management V2](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonapigatewaymanagementv2.html)

これはあくまで推測ですが、サービスリリース後に追加された大規模な変更や機能は別ページで管理されていますが、実際には一つのサービスとして機能しているため、このような形式を取っていると考えられます。

なおアクションが最も多かったサービスプレフィックスの上位5位は以下の通りです。
ec2が多いことは予想していましたが、chimeやconnectのように個人的に全く知らないサービスのアクションが予想以上に多いことが意外でした。

| サービスプレフィックス    | アクション総数   | ドキュメントURL                                                                              |
|:------------------------|:---------------|:---------------------------------------------------------------------------------------------|
| ec2                     | 631            | https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonec2.html       |
| sagemaker               | 347            | https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonsagemaker.html |
| chime                   | 309            | https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonchime.html     |
| iot                     | 271            | https://docs.aws.amazon.com/service-authorization/latest/reference/list_awsiot.html          |
| connect                 | 247            | https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonconnect.html   |

<a id="#Chapter3"></a>

# 3.最後に
以上、IAMポリシーで設定可能なアクションの総数に関するお話でした、
最後までお読みいただきありがとうございます。

お察しの通りこれは実務では役立たない無駄な知識ですが、
雑談などで`「IAMポリシーで設定できるアクションって16,386あんねん」`みたいに話題にしていただければ幸いです。

また記事中のコードに誤りや、
抽出しているアクションに不足や余分がある場合はご指摘お待ちしております..。
