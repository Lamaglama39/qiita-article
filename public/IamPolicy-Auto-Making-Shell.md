---
title: AWS CLIの実行結果から不足している権限をIAMポリシーに自動で追加したい
tags:
  - AWS
  - IAM
  - aws-cli
  - Bash
  - ShellScript
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- 発端や概要を記載 -->
# はじめに
皆さんがAWSを利用していて、苦痛を感じるのはどのようなときでしょうか？
あくまで個人の意見ですが、私が最も苦痛を感じるのはIAMポリシーの権限設定を検証している時です。

操作対象のリソースごとに公式ドキュメントを確認して権限を設定し、
実行してみてエラーが出たら不足している権限を調査し、権限を追加したらまた実行して...。
この一連の作業を何度も繰り返していると、賽の河原感が強いなぁと感じてしまいます。
しかも、基本的にどのサービスも使用する上で避けて通れないため、遭遇する頻度が高いことも辛いポイントです。

そこで今回、`「AWS CLIの実行結果に応じて、権限が不足しているアクションを自動でIAMポリシーに追加してくれるスクリプト」`を作成したので共有します。
詳細は後述しますが、現時点でいくつかの問題点があるため実用性は限られていますが、何かの参考になればと思います。

:::note info
本記事のスクリプトは以下リポジトリにまとめているので、実行する場合はこちらをご利用ください。
またスクリプト実行にあたり必要となるリソース作成用のTerraformも含んでいるため、合わせてご利用ください。
* [iam-action-auto-add](https://github.com/Lamaglama39/iam-action-auto-add)
:::


<!-- 各チャプター -->
<a id="#Chapter1"></a>

# 1.想定している利用シーン
このスクリプトの利用を想定しているシーンですが、
以下のように`アイデンティティベースのIAMポリシーを用いてAWS CLIを利用する場合`を想定しています。

```
(1) ローカルPCからIAMユーザー/IAMロールを利用してAWS CLIを実行する場合
(2) EC2インスタンスからインスタンスプロファイルを利用してAWS CLIを実行する場合
```

AWS CLIの実行結果をもとに権限を追加するという構成のため、
シェルスクリプト、またAWS CLIを実行できない環境ではこのスクリプトは使用できません。

またその他のポリシータイプの詳細については、以下のドキュメントをご参照ください。
* [IAM でのポリシーとアクセス許可](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/access_policies.html)


<a id="#Chapter2"></a>

# 2.処理内容
本スクリプトは大まかに以下の処理を実行しており、
実行するコマンドの数だけ、以下(2)～(6)を繰り返します。

```text:処理内容
(1).環境変数(env.sh)、コマンドリスト(command.txt)の読み込み
(2).コマンドリストから1行ずつコマンドを実行
    ➤ 正常終了した場合は、(2)で次のコマンドを実行
    ➤ エラーが発生した場合は、(3)以降を実行
(3).コマンド実行時のエラーメッセージから不足している権限を取得
    ➤ 対象の権限が存在しない場合はスキップし、(2)で次のコマンドを実行
(4).現行のIAMポリシーを取得
    ➤ 対象の権限が既に付与されている場合はスキップし、(2)で次のコマンドを実行
(5).IAMポリシーが5世代以上あればバージョニングを実施
(6).IAMポリシーを更新して、(2)でコマンドを再実行
```

## 環境変数
`env.sh`ではAWS CLIを実行するIAMユーザー、また権限追加対象のIAMポリシーを設定します。
各自の環境に合わせて設定値を記載してください。

```bash:env.sh
#!/bin/bash

#IAM認証設定
export AWS_ACCESS_KEY_ID="IAMユーザーのアクセスキーID"
export AWS_SECRET_ACCESS_KEY="IAMユーザーのシークレットアクセスキーID"

#アクション権限 設定対象
export TARGET_IAM_POLICY_ARN="権限追加対象IAMポリシーのARN"

```

## コマンドリスト
`command.txt`は実行するコマンドを記載します、
以下は例なので、各自実行したいコマンドを記載してください。
また複数行には対応していないので、`必ず1行で1つのコマンド`としてください。

```text:command.txt
aws iam list-users
aws ec2 describe-instances
aws rds describe-db-instances

```

## アクション 一覧ファイル
AWS CLIの実行が失敗し権限を追加する際に、追加対象のアクションが実際に存在するものであるかを確認するため、
全サービスの全アクションを記載した`iam_actions.txt`を参照して正常性を確認しています。

なお全アクションの取得方法は以下の記事で解説しているので、よろしければご参照ください。

* [【AWS】IAMポリシーで設定できるアクションがいくつあるか、あなたは知っていますか？](https://qiita.com/Lamaglama39/items/a096556e1ff8c08defc6)

## メイン処理
`add_action.sh`にて読み込んだコマンドリスト(command.txt)の行数だけ、コマンド実行と権限追加を繰り返します。
また具体的な処理の部分は`function.sh`に切り出しています。

なお実行結果の標準出力/標準エラー出力については、
画面とログファイル(OUTPUT_FILE)の両方に出力します。

```bash:add_action.sh
#!/bin/bash
set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "${0}")";pwd)"

#変数設定
source "${SCRIPT_DIR}/env.sh"
source "${SCRIPT_DIR}/function.sh"

#実行対象 コマンドリスト
export COMMAND_LIST="${SCRIPT_DIR}/command.txt"
export IAM_POLICY_JSON="${SCRIPT_DIR}/policy_document.json"

#実行結果 ログfile
declare OUTPUT_FILE
OUTPUT_FILE="${SCRIPT_DIR}/output_log_$(date +"%Y%m%d%H%M").txt"
export OUTPUT_FILE

#ログファイル出力設定
exec > >(tee -a "${OUTPUT_FILE}") 2>&1

# main処理
function main() {
	while read -r command; do
		while true; do

			#コマンド実行
      log_output "RunCommand : ${command}"
			if ! response=$(eval "${command}" --output text 2>&1 | sed -z 's/\n//g'); then

        #権限不足エラーが発生した場合、エラーメッセージを出力
				log_output "ErrorMessage : ${response}"

				#不足権限 取得
				target_action=$(printf "%s" "$response" | awk -F"perform: " '{print $2}' | awk '{print $1}' | sed -z 's/\n//g')

				#該当のアクションが存在するか確認
				if [[ -z "${target_action}" ]]; then
					log_output "Targetaction is empty."
					break
				elif grep -q "${target_action}" "data/iam_actions.txt"; then
					log_output "AddTargetAction : ${target_action}"
				else
					log_output "Unknown Action : ${target_action}"
					break
				fi

				# IAMポリシー取得
				get_iam_policy

				# IAMポリシーチェック処理
        if ! check_iam_policy "${target_action}"; then
          break
        fi

				# IAMポリシー バージョニング処理
				versioning_iam_policy

				# IAMポリシー 更新処理
				update_iam_policy "${target_action}"

			else
				##コマンドが成功した場合
				log_output "CommandExec : Success."
				break
			fi
		done
	done < "${COMMAND_LIST}"
}

main

```
```bash:function.sh
#!/bin/bash

# ログフォーマット
function log_output() {
	local -r log="$1"
	echo "${log}" | awk '{ print strftime("[%Y-%m-%d %H:%M:%S]"), $0 }'
}

# IAMポリシー 取得処理
# 現在のIAMポリシーをファイル出力
function get_iam_policy() {
	aws iam get-policy-version \
		--policy-arn "${TARGET_IAM_POLICY_ARN}" \
		--version-id "$(aws iam get-policy \
		--policy-arn "${TARGET_IAM_POLICY_ARN}" | jq -r '.Policy.DefaultVersionId')" | jq -r '.PolicyVersion.Document' > policy_document.json

	log_output "GetPolicy : ${TARGET_IAM_POLICY_ARN}"
}

# IAMポリシー 既存権限チェック
# 追加対象の権限がすでに付与されている場合は処理をスキップ
function check_iam_policy() {
  local -r add_action=$1

	if grep -q "${add_action}" "${IAM_POLICY_JSON}" ; then
		log_output "Action is already included in the permission."
		return 1
	fi
}

# IAMポリシー バージョニング処理
# バージョンが5以上であれば、もっとも古いものを削除する
function versioning_iam_policy() {
	local -r version_count=$(aws iam list-policy-versions \
														--policy-arn "${TARGET_IAM_POLICY_ARN}" \
														--query 'Versions[].VersionId' \
														--output text | wc -w)

	if [[ "$version_count" -ge 5 ]]; then
		local -r old_version=$(aws iam list-policy-versions \
														--policy-arn "${TARGET_IAM_POLICY_ARN}" \
														--query 'Versions[].VersionId' \
														--output text | awk '{print $NF}')

		aws iam delete-policy-version \
			--policy-arn "${TARGET_IAM_POLICY_ARN}" \
			--version-id "${old_version}"

		log_output "VersioningPolicy : Success."
	fi
}

# IAMポリシー 更新処理
function update_iam_policy() {
  local -r add_action=$1

	#不足している権限をファイルに追記
	jq --arg add_action "${add_action}" '.Statement[0].Action += [$add_action]' policy_document.json > temp.json && mv temp.json policy_document.json

	#追記したファイルをもとにIAMポリシーを更新する
	aws iam create-policy-version \
		--policy-arn "${TARGET_IAM_POLICY_ARN}" \
		--policy-document file://"${IAM_POLICY_JSON}" \
		--set-as-default \
		> /dev/null

	log_output "UpdatePolicy : Success."
	log_output "Waiting for update for 10 seconds."

	sleep 10
}

```

<a id="#Chapter3"></a>

# 3.問題点
ここまでお読みいただいた方はお気づきかもしれませんが、現在のスクリプトにはいくつかの問題点が存在します。
今後、これらの問題に対して改修を予定しています...。

その他にも改修すべき点や改善提案などがございましたら、
本記事のコメント欄やGitHubでのご連絡をお待ちしております。

## (1) AWS CLIの利用が前提となる
前述の通り、このスクリプトはAWS CLIを利用する環境を前提としています。
そのためAWS SDKなどでは使用できません。
これは今後の改善点なので、続報をお待ちください...。

## (2) エラーメッセージに対象のアクションが含まれていないと権限追加ができない
権限を追加する際はAWS CLI実行時のエラーメッセージに含まれる、
`サービスプレフィックス:アクション`の文字列を抽出しIAMポリシーへ反映しています。
例えば以下のエラーメッセージの場合、`iam:ListUsers`が抽出対象となります。
```text:エラーメッセージの例
An error occurred (AccessDenied) when calling the ListUsers operation: User: <AWS CLIを実行したIAMユーザー> is not authorized to perform: iam:ListUsers on resource: <AWS CLI実行対象のリソース> because no identity-based policy allows the iam:ListUsers action
```

そのためエラーメッセージに`サービスプレフィックス:アクション`の文字列が含まれていない場合、権限の追加が行えません。
例としてはS3を対象としたAWS CLI(aws s3)では権限の追加を行えません。
```text:S3 エラーメッセージ例
An error occurred (AccessDenied) when calling the ListBuckets operation: Access Denied
```

この部分はAWS CLIのエラーメッセージの形式が統一されない限りは、解消が難しい見込みです。

## (3) Actionの追記しか対応していない
現状のスクリプトはIAMポリシーへの`Action`追加のみ対象としており、
`Effect`、`Principal`、`Resource`、`Condition`などのIAMポリシーの他の要素には対応していません。
これも今後の改善点なので、続報をお待ちください...。


<a id="#Chapter4"></a>

## 参考ドキュメント

* [IAM でのポリシーとアクセス許可](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/access_policies.html)
* [AWS CLI Command Reference](https://docs.aws.amazon.com/ja_jp/cli/latest/index.html)
* [Service Authorization Reference](https://docs.aws.amazon.com/service-authorization/latest/reference/reference_policies_actions-resources-contextkeys.html)
