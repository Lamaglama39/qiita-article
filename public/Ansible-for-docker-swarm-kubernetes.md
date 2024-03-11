---
title: Ansibleで構築するコンテナオーケストレーション環境
tags:
  - Ubuntu
  - Docker
  - Ansible
  - kubernetes
private: false
updated_at: '2024-03-09T02:49:24+09:00'
id: 8d3cbd398d19644ec2a9
organization_url_name: null
slide: false
ignorePublish: false
---

<!-- 発端や概要を記載 -->
# はじめに

自宅でコンテナをいじるにあたり、
一台一台丹精込めてセットアップするのがめんどくさすぎてAnsibleでセットアップをまとめたので共有します。
`インストール～クラスター作成とノード追加`まで一括で実行できるので、
すぐにコンテナオーケストレーション環境を整えられます。

あえてマネージドサービス(EKS,AKS,GKE...etc)を利用せず、
おうちKubernetesをやる際などに参考にしていただければ幸いです。

:::note info
本記事は以下リポジトリで作成したプレイブックの解説を中心とした補足資料です。
すぐに構築したい方はリポジトリのREADMEを参考にして実行してください。
* [ansible-container-recipes](https://github.com/Lamaglama39/ansible-container-recipes/tree/main)
:::

<!-- 各チャプター -->
<a id="#Chapter1"></a>

# 1.実行環境
新しめのUbuntuを使いたかったので、今回は`Ubuntu23.04`を使用していますが、
基本的に`Ubuntu18.04以降`であれば問題なく動作します。

それ以上に古いバージョンのUbuntuや、
その他Debian系のディストリビューションでも利用可能ではありますが、
動作確認はしていないので各自でプレイブックの修正が必要になる可能性があります。

* Ansible コントロールノード

| 名称            | バージョン  | 説明  |
|:----------------|:-----------|:------|
|	Ubuntu          | 23.04      | わりと新しめのバージョンで、コードネームになっている⁠Lunar Lobster⁠（月のロブスター）ちゃんがキュートでGoodです。                |
|	Ansible-core    | 2.15.9     | 構成管理、アプリケーションデプロイメント、タスク自動化など、ITインフラの自動化をするツールです、エージェントレスなのが素敵です。  |
|	Python          | 3.9.1      | Ansibleが利用しているPythonです。                                                                                        |

* 管理対象ノード (Docker Swarm)

| 名称            | バージョン  | 説明  |
|:----------------|:-----------|:------|
|	Ubuntu          | 23.04      | 同上  |
|	docker-ce       | 25.0.2     | Dockerエンジンのコミュニティーエディションです。                                                                     |
|	docker-ce-cli   | 25.0.2     | DockerCLI のコミュニティーエディションです。 イメージのビルド/コンテナの起動・停止/コンテナの状態確認などの操作ができます。|
|	containerd.io   | 1.6.28     | Dockerコンテナランタイムのための標準のコアコンテナランタイムです。                                                     |
|	buildx          | 0.12.1     | Dockerのビルド拡張機能で、マルチアーキテクチャビルド、ビルドキットの使用などを提供します。                               |
|	compose         | 2.24.5     | 複数のコンテナを定義し、実行するためのツールです。一括でコンテナを管理することができます。                                |

* 管理対象ノード (Kubernetes)

| 名称            | バージョン  | 説明  |
|:----------------|:-----------|:------|
|	Ubuntu          | 23.04      | 同上  |
|	docker-ce       | 25.0.2     | 同上  |
|	docker-ce-cli   | 25.0.2     | 同上  |
|	containerd.io   | 1.6.28     | 同上  |
|	kubeadm         | 1.29.2     | Kubernetesクラスタを初期構築するためのツールです。クラスタの初期化/ノードの追加/クラスタのアップグレード/クラスタの設定/および削除などの操作が容易にできます。 |
|	kubelet         | 1.29.2     | Kubernetesクラスターの各ノードで実行されるコンポーネントの一つです。ノードを監視し、必要に応じてコンテナの起動、停止、および再起動を行います。                |
|	kubectl         | 1.29.2     | Kubernetesクラスターと対話するためのコマンドラインツールです。クラスターの管理、リソースの展開、監視、および操作を行うためのインターフェースを提供します。     |
|	helm            | 3.14.2     | Kubernetesアプリケーションのパッケージ管理ツールです。複雑なKubernetesアプリケーションを簡単にデプロイし、管理することができます。                          |

:::note warn
上記バージョンは2024年2月時点のものです。
Kubernetes以外は基本的にバージョンを指定していないので、別のバージョンがインストールされる可能性があります。
:::

<a id="#Chapter2"></a>

# 2.プレイブック
必要に応じて、 `Docker Swarm` か `Kubernetes` を導入してください。
なおパッケージ/ネットワークセグメントが競合しないようプレイブックを作成しているので両方導入することもできますが、
検証目的以外ではあまり利点がないのでおすすめしません。

* [Docker Swarm プレイブック](https://github.com/Lamaglama39/ansible-container-recipes/tree/main/ansible/docker-setting)
* [Kubernetes プレイブック](https://github.com/Lamaglama39/ansible-container-recipes/tree/main/ansible/kubernetes-setting)

また最低限の冪等性は担保しているので、各プレイブックを連続で複数回実行しても問題は発生しないようになっています。
ただしあくまで開発環境での利用を前提としているため、もし本番運用で利用される場合は入念に検証してから利用してください。

## 2-1.Docker Swarm
Docker公式が開発および管理しているオープンソースのコンテナオーケストレーションツールです。
Kubernetes全盛期の今、実運用でDocker Swarmを使う場面はほとんどないかと思いますが、
比較的ハードルが低い(当社比)なので、コンテナ環境の入門として触ってみるのはありです。
というか個人的に開発環境としての利用であればDocker Swarmで十分だと思います。

#### (1) Docker インストール
Dockerのリポジトリ追加の際に`lsb_release`コマンドで動的にUbuntuのコードネームを取得することで、対応しているリポジトリを登録しています。

```yaml:01-docker-install.yml 抜粋
    - name: Get Ubuntu codename
      command: lsb_release -cs
      register: ubuntu_codename
      changed_when: false

    - name: Add the Docker apt repository
      lineinfile:
        path: /etc/apt/sources.list.d/docker.list
        line: deb [signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu {{ ubuntu_codename.stdout }} stable
        create: true
```

* [01-docker-install.yml](https://github.com/Lamaglama39/ansible-container-recipes/blob/main/ansible/docker-setting/01-docker-install.yml)


#### (2) Docker Swarm クラスター作成
`pod_network_cidr`変数 でポッドで利用するIPアドレスプールを指定して、
`leader_primary_ip`変数でクラスターを作成するマスターノードのIPを取得しています。
また`docker info --format "{{.Swarm.LocalNodeState}}"`でクラスターの状況を確認し、すでに作成済みの場合はスキップします。

```yaml:02-docker-swarm-create.yml 抜粋
  vars:
    pod_network_cidr: "10.0.0.0/16"

  tasks:
    - name: Primary IP Address
      set_fact:
        leader_primary_ip: "{{ ansible_facts['default_ipv4']['address'] }}"

    - name: Check if Docker Swarm is already initialized
      shell: docker info --format '{{ "{{.Swarm.LocalNodeState}}" }}'
      register: swarm_status
      changed_when: false

    - name: Initialize Docker Swarm
      shell: docker swarm init --default-addr-pool "{{ pod_network_cidr }}" --advertise-addr "{{ leader_primary_ip }}"
      register: swarm_init
      when: swarm_status.stdout != 'active'
```

* [02-docker-swarm-create.yml](https://github.com/Lamaglama39/ansible-container-recipes/blob/main/ansible/docker-setting/02-docker-swarm-create.yml)

#### (3) Docker Swarm ノード参加
クラスター作成を行ったマスターノードでトークンを発行し、
各マスターノード/ワーカーノードをクラスターに参加させ、マスターノードはDrain設定を行っています。
なおクラスターが存在しない場合はスキップします。

```yaml:03-docker-swarm-join.yml 抜粋
    - name: Check if Docker Swarm is already initialized
      shell: docker info --format '{{ "{{.Swarm.LocalNodeState}}" }}'
      register: swarm_status
      changed_when: false

    - name: Drain manager node
      shell: docker node update --availability drain {{ ansible_facts['hostname'] }}
      when: swarm_status.stdout == 'active'
      changed_when: false

    - name: Get Docker Swarm manager join-token
      shell: docker swarm join-token manager -q
      register: manager_join_token
      changed_when: false

    - name: Get Docker Swarm worker join-token
      shell: docker swarm join-token worker -q
      register: worker_join_token
      changed_when: false
```

* [03-docker-swarm-join.yml](https://github.com/Lamaglama39/ansible-container-recipes/blob/main/ansible/docker-setting/03-docker-swarm-join.yml)

#### (4) Docker Swarm クラスター削除
ワーカーノードでクラスターからの離脱、マスターノードでクラスターの削除を実施します。
なおクラスターが存在しない場合はスキップします。

```yaml:04-docker-swarm-reset.yml 抜粋
    - name: Check if node is part of a Swarm cluster
      shell: docker info --format '{{ "{{.Swarm.LocalNodeState}}" }}'
      register: swarm_status
      changed_when: false

    - name: Join swarm cluster as a manager
      shell: "docker swarm leave --force"
      when: swarm_status.stdout == 'active'
```

* [04-docker-swarm-reset.yml](https://github.com/Lamaglama39/ansible-container-recipes/blob/main/ansible/docker-setting/04-docker-swarm-reset.yml)


## 2-2.Kubernetes
今やデフファクトスタンダードのコンテナオーケストレーションツールです。
主要なクラウドプロバイダーではマネージドKubernetesサービスを提供しており、日常的に目にする機会が増えています。

* [AWS EKS](https://aws.amazon.com/jp/eks/)
* [GCP GKE](https://cloud.google.com/kubernetes-engine?hl=ja)
* [Azure AKS ](https://azure.microsoft.com/ja-jp/products/kubernetes-service)


しかしクラスターの構築や簡単なコンテナのデプロイメントだけでも、月に最低でも1万円程度のコストがかかるため、
正直なところ個人開発での利用は少しハードルが高いです。
だからといって一からマスターノードを設定するのも手間がかかります。

このような背景から、今回はプレイブックを用いて初期設定を行うに至りました。


#### (1) Kubernetes インストール
Kubernetes関連のパッケージはインストール後にバージョンを固定しています。
またその他前提となるOS関連の設定、およびcontainerdの初期設定を実施しています。

```yaml:01-kubernetes-install.yml 抜粋
    - name: Hold Kubernetes Packages
      dpkg_selections:
        name: "{{item}}"
        selection: hold
      with_items:
        - kubelet
        - kubeadm
        - kubectl

    - name: Check if containerd config file cri
      command: grep -q 'disabled_plugins = \[\]' /etc/containerd/config.toml
      register: cri_config_check
      failed_when: false
      changed_when: false
      ignore_errors: true

    - name: Create containerd default config file
      shell: containerd config default > /etc/containerd/config.toml
      when: cri_config_check.rc != 0

    - name: Enable cri
      replace:
        path: /etc/containerd/config.toml
        regexp: 'disabled_plugins\s*=\s*\["cri"\]'
        replace: 'disabled_plugins = []'
      when: cri_config_check.rc != 0

    - name: Check if containerd config file cgroup
      command: grep -q 'SystemdCgroup = true' /etc/containerd/config.toml
      register: cgroup_config_check
      failed_when: false
      changed_when: false
      ignore_errors: true
```

* [01-kubernetes-install.yml](https://github.com/Lamaglama39/ansible-container-recipes/tree/main/ansible/kubernetes-setting/01-kubernetes-install.yml)

#### (2) Kubernetes クラスター作成
`pod_network_cidr`変数 でポッドで利用するIPアドレスプールを指定して、
`leader_primary_ip`変数でクラスターを作成するマスターノードのIPを取得しています。
またクラスターが作成済みの場合はスキップします。

```yaml:02-kubernetes-create-cluster.yml 抜粋
  vars:
    pod_network_cidr: "10.1.0.0/16"

  tasks:
    - name: Primary IP Address
      set_fact:
        leader_primary_ip: "{{ ansible_facts['default_ipv4']['address'] }}"

    - name: Initialize the Kubernetes cluster
      shell: |
        kubeadm init --control-plane-endpoint={{ leader_primary_ip }}:6443 --pod-network-cidr={{ pod_network_cidr }} --upload-certs
      args:
        creates: /etc/kubernetes/admin.conf
```

* [02-kubernetes-create-cluster.yml](https://github.com/Lamaglama39/ansible-container-recipes/tree/main/ansible/kubernetes-setting/02-kubernetes-create-cluster.yml)

#### (3) Kubernetes ネットワーク作成
calicoを利用してネットワークを作成しています、また設定ファイルの修正もプレイブック内で実施しています。
なおcalico-configがすでに稼働している場合は、スキップします。

```yaml:03-kubernetes-create-network.yml 抜粋
    - name: Check if Calico manifest already exists
      stat:
        path: "./calico.yaml"
      register: calico_manifest

    - name: Download Calico manifest
      get_url:
        url: https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/calico.yaml
        dest: "./calico.yaml"
        force: false
      when: not calico_manifest.stat.exists


    - name: Check for IP_AUTODETECTION_METHOD marker in calico.yaml
      shell: "grep -q '# BEGIN ANSIBLE MANAGED BLOCK IP_AUTODETECTION_METHOD' ./calico.yaml"
      register: ip_autodetection_check
      failed_when: false
      changed_when: false
      ignore_errors: true

    - name: Add IP_AUTODETECTION_METHOD to calico.yaml
      blockinfile:
        path: "./calico.yaml"
        insertafter: 'value: "autodetect"'
        block: |2
                      - name: IP_AUTODETECTION_METHOD
                        value: "cidr={{ leader_primary_ip_cidr_result.stdout }}"
        marker: "# {mark} ANSIBLE MANAGED BLOCK IP_AUTODETECTION_METHOD"
      when: ip_autodetection_check.rc != 0

    - name: Check for CALICO_IPV4POOL_CIDR marker in calico.yaml
      shell: "grep -q '# BEGIN ANSIBLE MANAGED BLOCK CALICO_IPV4POOL_CIDR' ./calico.yaml"
      register: calico_ipv4pool_cidr_check
      failed_when: false
      changed_when: false
      ignore_errors: true

    - name: Add CALICO_IPV4POOL_CIDR to calico.yaml
      blockinfile:
        path: "./calico.yaml"
        insertafter: '#   value: "192.168.0.0/16"'
        block: |2
                      - name: CALICO_IPV4POOL_CIDR
                        value: "{{ pod_network_cidr }}"
        marker: "# {mark} ANSIBLE MANAGED BLOCK CALICO_IPV4POOL_CIDR"
      when: calico_ipv4pool_cidr_check.rc != 0

    - name: Check if Calico is already applied
      shell:
        cmd: kubectl get cm -n kube-system | grep -q calico-config
      register: calico_check
      failed_when: false
      changed_when: false
      ignore_errors: true
```

* [03-kubernetes-create-network.yml](https://github.com/Lamaglama39/ansible-container-recipes/tree/main/ansible/kubernetes-setting/03-kubernetes-create-network.yml)

#### (4) Kubernetes ノード参加
クラスター作成を行ったマスターノードでトークンを発行し、
各マスターノード/ワーカーノードをクラスターに参加させ、マスターノードはDrain設定を行っています。
なおクラスターが存在しない場合はスキップします。

```yaml:04-kubernetes-join.yml 抜粋
    - name: manager join-certs
      shell: kubeadm init phase upload-certs --upload-certs | tail -n 1
      become: true
      register: join_certs
      changed_when: false

    - name: worker join-token
      shell: kubeadm token create --print-join-command
      register: join_token
      changed_when: false

    - name: Check node status in the Kubernetes cluster
      shell: kubectl get nodes | grep {{ ansible_hostname }} || true
      register: node_check
      ignore_errors: true
      changed_when: false
      delegate_to: "{{ groups['master_leader'][0] }}"

    - name: Join Kubernetes cluster as a manager
      shell: "{{ hostvars[groups['master_leader'][0]]['join_token'].stdout }} --control-plane --certificate-key {{ hostvars[groups['master_leader'][0]]['join_certs'].stdout }}"
      become: true
      when: node_check.stdout == ""
```

* [04-kubernetes-join.yml](https://github.com/Lamaglama39/ansible-container-recipes/tree/main/ansible/kubernetes-setting/04-kubernetes-join.yml)

#### (5) Kubernetes クラスター削除
クラスターの削除、および関連するクラスターの設定ファイルを削除し、`iptables`の設定も削除しています。
なおクラスターにノードが存在しない場合はスキップします。

```yaml:05-kubernetes-reset.yml 抜粋
    - name: Reset kubeadm if cluster is configured
      shell: kubeadm reset --force
      when: kubectl_nodes.stderr == ""
      ignore_errors: yes

    - name: Stop kubelet if cluster is configured
      service:
        name: kubelet
        state: stopped
      when: kubectl_nodes.stderr == ""

    - name: Remove Kubernetes configuration directories if cluster is configured
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - ~/calico.yaml
        - /etc/kubernetes/
        - ~/.kube/
        - /var/lib/kubelet/
        - /var/lib/cni/
        - /etc/cni/
        - /var/lib/etcd/
      when: kubectl_nodes.stderr == ""

    - name: Clear iptables if cluster is configured
      shell: iptables -F && iptables -X
      ignore_errors: yes
      when: kubectl_nodes.stderr == ""
```

* [05-kubernetes-reset.yml](https://github.com/Lamaglama39/ansible-container-recipes/tree/main/ansible/kubernetes-setting/05-kubernetes-reset.yml)


<a id="#Chapter3"></a>

## 参考ドキュメント
プレイブックの作成に際して、以下のドキュメントおよび記事を参考にさせていただきました。
この場を借りてお礼申し上げます。

* [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
* [kubeadmのインストール](https://kubernetes.io/ja/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
* [Ubuntu 22.04 に Kubernetes をインストールして自宅クラウド](https://rabbit-note.com/2022/08/09/build-kubernetes-home-cluster/)
* [kubeadm で Kubernetes 1.26 クラスターを作るための下準備を Ansible でまとめてみた](https://zenn.dev/imksoo/articles/151adbea791b51)
* [kubeadmで作成したk8sクラスタをresetする時に気を付けること](https://qiita.com/ohtsuka-shota/items/0643614a3cf197ac55b6)
