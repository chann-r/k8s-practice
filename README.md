# k8s-practice

## このリポジトリの再現手順


## このリポジトリの動作確認


## k8sについて
### motivation
- アプリケーションはなるべく複数マシンに分けてデプロイし、障害が起きても安定稼働できるようにしたい
- アプリケーションごとに「このアプリケーションは負荷が高そうなので高スペックのマシンで稼働させたい」などマシンへのスペック要件が異なる

### 概要
k8sデプロイするとクラスターが展開される

k8sクラスターはコンテナ化されたアプリケーションを実行するノード（マシン）の集合のことで、ワーカーノードとマスターノードから構成される
  - ワーカーノードは、Pod（k8s上のデプロイの最小単位で、1つ以上のコンテナを持つ）をホストする
  - マスターノードは、クラスター内のワーカーノードとPodを管理する
<img width="1238" alt="スクリーンショット 2021-09-04 12 20 31" src="https://user-images.githubusercontent.com/53222150/132080790-31a7a88b-8cd1-486a-a9a8-d0800b0f09cb.png">

### マスターノードについて
5つのコンポーネントから構成される
- kube-apiserver

  kubeletとのやりとりなどクラスター内部や外部とのやり取りをするインターフェース
- etcd

  k8sクラスターの状態を保存するためのkey-valueストア
- kube-scheduler

　アプリケーションをどのワーカーノードで実行するのかスケジューリングする
- kube-controller-manager

　理想状態と現状を比較し、違いがある場合には現状に近づけようとする
- cloud-controller-manager

　cloud関連（ローカルだと触れない）

### ワーカーノードについて
2つのコンポーネントから構成される
- kubelet

　マスターノード（kube-apiserver）と連絡を取り、ワーカーノードのコンテナを停止したり起動したり、その情報をマスターノードに伝達する
- kube-proxy

　ネットワークルールをメンテナンス役割があり、これによってPodへのネットワーク通信が可能になる
 
 
## k8sリソース

### Namespace
初期Namespaceは以下の2つがある
- default：Namespaceを指定しない場合はここをに作成・参照
- kube-system：k8sのシステムによって作成されたオブジェクトのためのNamespace

### Pod

同一Pod内のコンテナは同じStorageにアクセスでき、IPアドレスとPortを含むネットワーク名前空間を共有している。

そのため、同一Pod内のコンテナ同士はlocalhostで通信できる。

### ReplicaSet
常に指定したレプリカ数のPodを保つ役割

Deploymentの中で管理する

フィールドは以下
- replicas：稼働させたいPodの数
- pod templates：Podを作成するためのテンプレート
- selector：対象とするPodを特定するためのもの（pod templates内のラベルと同じである必要がある）

### Deployment
Podの管理を人の代わりに自動でやってくれる

以下のような役割がある
- ReplicaSetの更新（ロールアウト）
- ロールバック
- スケールアップ・ダウン（ReplicaSetのreplica数変更）

フィールドは以下
- ReplicaSetと同じ
- strategy：更新戦略のことで、RecreateかRollingUpdate（デフォルト）を指定
  - ダウンタイムがあってはいけない時はRollingUpdateにする
    - maxUnavailable：更新処理中に利用不可になるPodの最大数
    - maxSurge：更新処理にエクストラで追加できるPodの最大数
- revisionHistory：過去のバージョンとしてReplicaSetを残しておく数
  - デフォルトは10
  - 残しているとロールバックできる
- paused：一時停止するかどうか
- progressDeadlineSeconds：更新の最大秒数（これ以上だと失敗したとみなされる）
  - デフォルトは600秒

### ConfigMap
機密性のないデータをキーと値のペアで保存し、Podから参照可能にすることでコンテナイメージと環境固有の設定を切り離すことができる

使い方①
- ConfigMap（yaml）の作成（dataプロパティ ）
- Podの設定でConfigMapから環境変数を読み込む（envFromプロパティで読み込む）

使い方②
- ConfigMap（Yaml）の作成
- Podの設定でConfigMapをボリュームとして読み込む
  - Podにvolumeを定義
  - コンテナにvolumeをマウント

### Secret
パスワードやトークンなどの機密情報を保存・管理し、Podから参照可能にするためのもの

ConfigMapと同様に、Pod内のコンテナにVolumeとしてマウントしたり、コンテナの環境変数として設定することが多い

フィールドは以下
- data：base64でエンコードされたものを記述（echo -n "aaa" | base64 などでエンコード）
- stringData：生データ

### Service
各PodはIPアドレスを持っていて、そのPodが作成や削除の度に変化するのでアプリケーションのIPアドレスが変化してしまう.
その変化を管理してくれる役割.

対象となるPodをSelectorで認識する.

Serviceを作成するとターゲットとなるPodのIPアドレスがEndpointsというリソースに格納される.

## yamlの書き方

### 必須フィールド
- apiVersion：どのバージョンのKubernetes APIか
- kind：どの種類のオブジェクトか（Pod, Deploymentなど）
- metadata：オブジェクトを一意に特定するための情報（name, UID, namespaceなど）
- spec：理想状態

### その他
- ハイフン3つ（---）を書くことでリソースを区切り、複数のリソースを書くことができる

## kubectlコマンド
kubectl（クーべコントロール）によって、yamlやコマンドをAPIリクエストに変えてkube-apiserverにHTTPリクエストを送る

k8sの設定ファイルは`$HOME/.kube/config`にあり、3つを設定している
- cluster：k8sクラスタ
- user：接続するユーザー
- context：clusterとuserの組み合わせ

コマンドの基本形
```
kubectl [実行したい操作] [リソースタイプ] [リソース名]
```

### 基本

#### 現在のコンテキストを表示する
```
kubectl config current-context
```

#### どのk8sクラスタに接続しているかを確認する
```
kubectl cluster-info
```

#### リソースの一覧取得
```
kubectl api-resources
```

#### リソースの取得（`-n`オプションなしだとdefaultというNamespaceを参照する）
```
kubectl get [リソースタイプ（PodとかDeploymentとか）] -n [Namespace]
```
※ `kubectl describe`もほぼ同じ

#### 全てのNamespaceのリソースを取得
```
kubectl get [リソースタイプ] --all-namespaces
```

#### リソースの作成・削除
```
kubectl create/delete -f [ファイル]
```
```
kubectl create/delete [リソースタイプ] [リソース名]
```

#### リソースの適用
```
kubectl apply -f [ファイル]
```

#### yamlの作成
```
kubectl create ns [Namespace] --dry-run=client -o yaml > <yamlファイル>
```

#### podに入る
```
kubectl exec -it <Pod名> sh
```

### Pod

#### Podの中で特定のimageを実行
```
kubectl run [Pod名] --image=[image名]
```

#### Podを監視
```
watch kebectl get pod
```

#### ログを出力
```
kubectl logs [Pod名] --follow -tail 20
```

#### ラベルを表示
```
kubectl get pod --show-labels
```

#### Selectorを指定して表示
```
kubectl get pod --selector app=MyApp
```

### ReplicaSet
```
kubectl scale rs/[Replicaset名] --replicas=[数]
```
※ Deploymentやyamlファイルで管理するので使わない

### Deployment

#### なんだろこれ
```
kubectl set image deployment/<DeploymentのName> <containerの名前>=<image>
```

#### ロールアウトできる履歴を確認
```
kubectl rollout history deployment.[apiVersion].apps/[Deployment名]
```

#### ロールアウトを実行
```
kubectl rollout undo deployment.[apiVersion].apps/[Deployment名] --to-revision=<リビジョンの数>
```

### Secret

#### エンコード
```
echo -n "aaa" | base64
```

#### デコード
```
echo -n "kdfljgalskjfa" | base64 --decode
```

#### Secretの作成
```
kubectl create secret generic [Secret名] --from-literal=[キー名]=[値]
```

### Service

#### endpointを確認
```
kubectl get endpoints [Service名]
```

#### ポートフォワーディング
```
kubectl port-forward svc/[Service名] [localのポート]:[serviceのポート]
```
