---
title: EC2のNameタグを指定してSSMセッションマネージャーで接続しよう。
tags:
  - AWS
  - EC2
  - SSM
private: false
updated_at: '2023-12-20T22:23:07+09:00'
id: 215fdf0ffaa08a1efe9b
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに
<!-- 発端や概要を記載 -->
セッションマネージャー、めっちゃくちゃ便利ですよね？
今更言うまでもないですが従来の煩雑なSSHの設定が不要となり、なおかつセキュリティーグループのSSH用ポートの穴あけも不要、
またアクセス権限をIAMに集約できるなど、優秀な点を挙げればきりがないです。

ただし唯一気に入らない点があるとすれば、
AWS CLIで接続する際に`インスタンスID`を指定する必要があるという点です。

**「インスタンスIDをいちいち調べるのも面倒くさいし、再作成したらID変わるし面倒くさすぎるだろ...。」**
と、思ったAWSユーザは全国に1億2000万人はいるでしょう。
そんなわけで、EC2のNameタグを指定してセッションマネージャーで接続するシェルスクリプトを共有します。

<!-- 各チャプター -->
<a id="#Chapter1"></a>

# 1.シェルスクリプト

結論ですが、以下のスクリプトを丸コピして実行してください。
使い方は後述の項目を参照してください。

```bash:select-session.sh
#!/bin/bash

# get InstanceID function
get_instance_id_by_name() {
  aws ec2 describe-instances \
    --filter "Name=tag-key,Values=Name" \
              "Name=tag-value,Values=$1" \
    --query 'Reservations[*].Instances[*].InstanceId' \
    --output text
}

# get all InstanceID and NameTag function
get_all_instances() {
  aws ssm describe-instance-information \
    --query 'InstanceInformationList[*].[InstanceId]' \
    --output text | while read -r id; do
        name=$(aws ec2 describe-instances \
                --instance-ids "$id" \
                --query 'Reservations[*].Instances[].Tags[?Key==`Name`].Value' \
                --output text)
        echo "$name  ($id)"
      done
}

# main
case $# in
  1)
    InstanceID=$(get_instance_id_by_name "$1")
    aws ssm start-session --target "$InstanceID"
    ;;

  0)
    mapfile -t instances < <(get_all_instances)
    PS3='Select Instance: '
    select instance in "${instances[@]}"; do
      [[ -n $instance ]] || continue
      InstanceID=$(echo "$instance" | awk '{print $2}')
      aws ssm start-session --target "$InstanceID"
      break
    done
    ;;

  *)
    echo "Error: Invalid number of arguments."
    echo "Usage1 Specify Name tag: $0 <NameTag>"
    echo "Usage2 Select from list: $0"
    exit 1
    ;;
esac
```

<a id="#Chapter2"></a>

# 2.使用例

引数にNameタグを指定して実行した場合は、対象にそのまま接続します。

```text:Nameタグを指定した場合
$ select-session.sh <NameTag>

Starting session with SessionId: XXXXXXX-XXXXXXXXXXXXX
sh-4.2$ 
```

引き数なしで実行した場合は、
セッションマネージャーで接続可能なEC2が表示されるので、接続したいEC2の番号を入力してください。

```text:Nameタグを指定しない場合
$ select-session.sh
1) ec2-web-A i-AAAAAAAAAAAAAAAAA
2) ec2-ap-B i-BBBBBBBBBBBBBBBBB
3) ec2-db-C i-CCCCCCCCCCCCCCCCC
Select Instance: 3

Starting session with SessionId: XXXXXXX-XXXXXXXXXXXXX
sh-4.2$
```


<a id="#Chapter3"></a>

# 3.解説

引き数でNameタグを指定した場合は、
Nameタグをもとに`aws ec2 describe-instances`を実行して、Nameタグに紐づくインスタンスIDを取得します。
その後、取得したインスタンスIDを指定して`aws ssm start-session`を実行します。

```bash:Nameタグを指定した場合
# get InstanceID function
get_instance_id_by_name() {
  aws ec2 describe-instances \
    --filter "Name=tag-key,Values=Name" \
              "Name=tag-value,Values=$1" \
    --query 'Reservations[*].Instances[*].InstanceId' \
    --output text
}

---------------------------------------------------------

  1)
    InstanceID=$(get_instance_id_by_name "$1")
    aws ssm start-session --target "$InstanceID"
    ;;
```

次にNameタグを指定しない場合ですが、
`aws ssm describe-instance-information`でSSMセッションマネージャーで接続可能なインスタンスIDを取得します。
取得したインスタンスIDをもとに`aws ec2 describe-instances`を実行して、NameタグとインスタンスIDを出力します。

その後NameタグとインスタンスIDを配列に格納して、
`select`で任意のEC2を選択できるようにしています。

```bash:Nameタグを指定しない場合
# get all InstanceID and NameTag function
get_all_instances() {
  aws ssm describe-instance-information \
    --query 'InstanceInformationList[*].[InstanceId]' \
    --output text | while read -r id; do
        name=$(aws ec2 describe-instances \
                --instance-ids "$id" \
                --query 'Reservations[*].Instances[].Tags[?Key==`Name`].Value' \
                --output text)
        echo "$name  ($id)"
      done
}

---------------------------------------------------------

  0)
    mapfile -t instances < <(get_all_instances)
    PS3='Select Instance: '
    select instance in "${instances[@]}"; do
      [[ -n $instance ]] || continue
      InstanceID=$(echo "$instance" | awk '{print $2}')
      aws ssm start-session --target "$InstanceID"
      break
    done
    ;;
```

ただし上記はインスタンスIDごとに`aws ec2 describe-instances`を実行しているため、
実行時間がめちゃ遅いです...。
具体的にはコマンドの実行結果を取得できるまで約1秒かかるので、
単純計算でアカウント内にEC2が10台あれば`select`できるまで10秒かかります。

というわけで、何か改良案があればご連絡ください...。
