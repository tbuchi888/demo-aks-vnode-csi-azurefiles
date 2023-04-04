---
title: AKS 仮想ノード( ACI ) 上の Pod から Azure ファイル共有をマウントする（ CSI 版 ）
tags: Azure AKS virtual-kubelet kubernetes
author: tbuchi888
slide: false
---
# 0. はじめに
Azure Kubernetes Service (以降 AKS ) は
* クラスター内の Worker node 数（ node pool ）を、需要に応じて、設定範囲内で自動で増減可能なオートスケーラー機能に加えて
* クラスターの許容量を超える急なバーストトラフィック等への対応として、OSS の Kubernetes kubelet の実装である、CNCF の [Virtual Kubelet](https://www.cncf.io/projects/virtual-kubelet/) を利用して、仮想ノードとして Azure Container Instances (以降 ACI )上に Pod をデプロイ可能です。

今回、ACI 仮想ノード上の複数 Pod より、ReadWriteMany 可能な Azure Files の共有フォルダ（ Azure ファイル共有）をマウントする際のハマリポイントなどを記事にしました。
結論からいいますと、ACI 仮想ノード上でCSI ストレージドライバーを利用する場合でも、以前のazurefileのストレージクラスと同じように PV （ Persitent volume ）、PVC （ Persistent Volume Claim ） の利用はできませんので、静的にボリュームを作成し、Secretとして、インラインボリュームとしてマウントしてあげる必要があります。

なお、通常の AKS ノード（ Virtual Machine Scale Sets による node pool ）上の Pod から、Azure ファイル共有のマウントについては PV （ Persitent volume ）、PVC （ Persistent Volume Claim ） を利用した、静的プロビジョニングや動的プロビジョニングなどは、AKSクラスタのバージョンによる考慮事項は多少あるものの、Kubernetes のスキルがあれば、特に問題なく利用できるのではと思います。

:::note 
* 本記事は、以下過去に執筆した記事を、CSI ストレージドライバーを利用する形で再度検証を行い ReWrite したものとなります。
  * https://qiita.com/tbuchi888/items/cf82d0d9ab7f8bb47ec9
* [ドキュメント修正のPR＃106637](https://github.com/MicrosoftDocs/azure-docs/pull/106637)が、先日無事マージされ、日本語版公式ドキュメントにも反映されましたので、このタイミングであらためて記事を書きます。
* なお、実行結果などは執筆時点（2023/04/04）の内容です
:::


# 1. 実行環境について
本記事は、以下 AKS バージョンで試しています。

```
$ kubectl get node 
NAME                                STATUS   ROLES   AGE   VERSION
aks-agentpool-19611256-vmss000002   Ready    agent   25h   v1.24.9
virtual-node-aci-linux              Ready    agent   25d   v1.19.10-vk-azure-aci-1.4.8
```

# 2. 事前準備と下調べ
+ AKS で仮想ノードを有効化します。
    + 仮想ノードを有効化するには、AKS クラスタで`Azure CNI` ネットワーク プラグインを利用する必要があります。
    + 本記事では有効化手順の説明をしませんが、詳細は以下を参考にしてください。
    + [az コマンドでの CLI](https://docs.microsoft.com/ja-jp/azure/aks/virtual-nodes-cli)でも可能ですが、Azure Portalより [AKS クラスタ作成時に有効化する](https://docs.microsoft.com/ja-jp/azure/aks/virtual-nodes-portal)方法が楽だと思います。
+ 仮想ノード使用にあたって
    + 仮想ノード上へ Pod をデプロイするにはいくつか設定が必要（後述）
    + AKS での仮想ノード使用時の制限事項がいくつかある（後述）

# 3. 仮想ノード上へ Pod をデプロイするにはいくつか設定が必要
仮想ノードの[サンプルアプリ](https://docs.microsoft.com/ja-jp/azure/aks/virtual-nodes-portal#deploy-a-sample-app)を確認すると、[toleration](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) と[nodeSelector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) の定義がされていることが分かります。

https://docs.microsoft.com/ja-jp/azure/aks/virtual-nodes-portal#deploy-a-sample-app

https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/

https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/

* 仮想ノードへの Pod スケジューリングを許可するために、`tolerations`を設定する必要がある
    * 重要な Pod が意図せずに仮想ノードへスケジュールされないように 仮想ノードには[Taints](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)が設定されているため、`tolerations`により回避する設定を追加
* さらに、`nodeSelector`を設定することで、仮想ノード上に限定して Pod を動かすことが可能
:::note 
なお、このサンプルは動作確認テストのために、明示的に`nodeSelector`を定義して仮想ノード上で Pod が起動するようにしていますが
バースト時のみ、仮想ノードへ Pod をスケジューリングしたい場合は、`nodeSelector`を定義しないようにします
:::

``` aci-sample-deploy.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aci-helloworld
spec:
  replicas: 1
  selector:
    matchLabels:
      app: aci-helloworld
  template:
    metadata:
      labels:
        app: aci-helloworld
    spec:
      containers:
      - name: aci-helloworld
        image: mcr.microsoft.com/azuredocs/aci-helloworld
        ports:
        - containerPort: 80
      nodeSelector:
        kubernetes.io/role: agent
        beta.kubernetes.io/os: linux
        type: virtual-kubelet
      tolerations:
      - key: virtual-kubelet.io/provider
        operator: Exists
```


# 4. Azure ファイル共有に関する、仮想ノード使用時の制限事項がある
通常の AKS ノードと比べて、仮想ノードを利用する場合、いくつか制限事項があり、ファイル共有関連で以下の制限がありますので、そちらを参考に試してみます。
その他、制限事項の詳細はこちらのドキュメントを参照。

https://learn.microsoft.com/ja-jp/azure/aks/virtual-nodes#known-limitations

> Azure Files 共有のサポート[汎用 V2](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-account-overview#types-of-storage-accounts) と[汎用 V1 ](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-account-overview#types-of-storage-accounts)をマウントするボリューム。 ただし、仮想ノードでは現在、[永続ボリューム](https://learn.microsoft.com/ja-jp/azure/aks/concepts-storage#persistent-volumes)と[永続ボリューム要求](https://learn.microsoft.com/ja-jp/azure/aks/concepts-storage#persistent-volume-claims)がサポートされていません。 [Azure Files 共有を含むボリュームをインライン ボリュームとして](https://learn.microsoft.com/ja-jp/azure/aks/azure-csi-files-storage-provision#mount-file-share-as-an-inline-volume)マウントする手順に従います。

:::note 
こちらが、今回のポイントとなる制限事項です
仮想ノードではファイル共有の PV （ Persitent volume ）、PVC （ Persistent Volume Claim ）としてのマウントをサポートしていません
インライン ボリュームとしてマウントする必要があります！
:::

## 4.1 Azure ファイル共有の事前準備
それでは、[Azure ファイル共有を作成する](https://learn.microsoft.com/ja-jp/azure/aks/azure-csi-files-storage-provision#create-an-azure-file-share)に従ってファイル共有のマウント準備をしていきます。

https://learn.microsoft.com/ja-jp/azure/aks/azure-csi-files-storage-provision#create-an-azure-file-share

### 4.1.1. Azure ファイル共有を作成する
* AKS_PERS_XXX の環境変数の値を自身の環境に置き換えて実行します。
* Azure portal からの作成ももちろん可能ですが、次の Step で、Azure ファイル共有用の Kubernetes の Secret を作成する必要があるため、az コマンドの方が、試しやすいかと思います。

```
# Change these four parameters as needed for your own environment
AKS_PERS_STORAGE_ACCOUNT_NAME=mystorageaccount$RANDOM
AKS_PERS_RESOURCE_GROUP=myAKSShare
AKS_PERS_LOCATION=eastus
AKS_PERS_SHARE_NAME=aksshare

# Create a resource group
az group create --name $AKS_PERS_RESOURCE_GROUP --location $AKS_PERS_LOCATION

# Create a storage account
az storage account create -n $AKS_PERS_STORAGE_ACCOUNT_NAME -g $AKS_PERS_RESOURCE_GROUP -l $AKS_PERS_LOCATION --sku Standard_LRS

# Export the connection string as an environment variable, this is used when creating the Azure file share
export AZURE_STORAGE_CONNECTION_STRING=$(az storage account show-connection-string -n $AKS_PERS_STORAGE_ACCOUNT_NAME -g $AKS_PERS_RESOURCE_GROUP -o tsv)

# Create the file share
az storage share create -n $AKS_PERS_SHARE_NAME --connection-string $AZURE_STORAGE_CONNECTION_STRING

# Get storage account key
STORAGE_KEY=$(az storage account keys list --resource-group $AKS_PERS_RESOURCE_GROUP --account-name $AKS_PERS_STORAGE_ACCOUNT_NAME --query "[0].value" -o tsv)

# Echo storage account name and key
echo Storage account name: $AKS_PERS_STORAGE_ACCOUNT_NAME
echo Storage account key: $STORAGE_KEY
```

### 4.1.2. Kubernetes の Secret を作成する
* Azure ファイル共有用へアクセスするために、Kubernetes の Secret を作成します。
* 前の Step の実行結果である `AKS_PERS_STORAGE_ACCOUNT_NAME` と `STORAGE_KEY` 環境変数を利用していますが、 既存の Azure ストレージアカウントや、Portal か ら作成した場合には、ストレージアカウント名とアクセスキーを設定します。
  
```
kubectl create secret generic azure-secret --from-literal=azurestorageaccountname=$AKS_PERS_STORAGE_ACCOUNT_NAME --from-literal=azurestorageaccountkey=$STORAGE_KEY
```

以上で、ファイル共有を AKS で利用する準備が整いました。

## 4.2 仮想ノードはファイル共有の PV （ Persitent volume ）、PVC （ Persistent Volume Claim ）マウントをサポートしていない
::: note
__結論から言うと、こちらの方法がハマりポイントです__
なんども言いますが、公式ドキュメントの制約事項にも、加筆した通り
* 以下の PV （ Persitent volume ）、PVC （ Persistent Volume Claim ）利用は、通常の AKS ノード上ではうまくいくのですが、仮想ノード上では利用できませんのでご注意ください
* 以下は仮想ノード上でのマウントエラーのナレッジのために記載してているものとなります
:::

ファイル共有を永続ボリューム（ Persitent volume ）としてマウントするために
前の Step で作成した、ファイル共有用の Secret を使って PV （ Persitent volume ）、PVC （ Persistent Volume Claim ）を静的に作成します。
`resourceGroup`や`shareName`、`nodeStageSecretRef`などは、前述の 4.1. の内容に置き換えてください。

``` pv-pvc-csi.yml
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/provisioned-by: file.csi.azure.com
  name: azurefile
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: azurefile-csi
  csi:
    driver: file.csi.azure.com
    readOnly: false
    volumeHandle: unique-volumeid  # make sure this volumeid is unique for every identical share in the cluster
    volumeAttributes:
      resourceGroup: myAKSShare  # optional, only set this when storage account is not in the same resource group as node
      shareName: aksshare
    nodeStageSecretRef:
      name: azure-secret
      namespace: default
  mountOptions:
    - dir_mode=0777
    - file_mode=0777
    - uid=0
    - gid=0
    - mfsymlinks
    - cache=strict
    - nosharesock
    - nobrl
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azurefile
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile-csi
  volumeName: azurefile
  resources:
    requests:
      storage: 5Gi
```

kubectl コマンドで AKS へ apply します。

```
kubectl apply -f pv-pvc.yml
```

作成後に、PV と PVC が以下のような状態になっていれば成功です。

```
$ kubectl get pv,pvc
NAME                         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS    REASON   AGE
persistentvolume/azurefile   5Gi        RWX            Retain           Bound    default/azurefile   azurefile-csi            13s

NAME                              STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS    AGE
persistentvolumeclaim/azurefile   Bound    azurefile   5Gi        RWX            azurefile-csi   12s
```

次に、以下 Deployment を使って Pod を仮想ノードへデプロイします。

``` dp-azurefiles-pvc-vol-acinode.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-azurefiles-pvc-acinode
  labels:
    node: aci
    app: nginx-azurefiles-pvc-acinode
spec:
  minReadySeconds: 30
  strategy:
    type: RollingUpdate
  replicas: 4
  selector:
    matchLabels:
      node: aci
      app: nginx-azurefiles-pvc-acinode
  template:
    metadata:
      name: nginx-azurefiles-pvc-acinode
      labels:
        app: nginx-azurefiles-pvc-acinode
        node: aci
    spec:
      containers:
      - image: docker.io/nginx:1.20
        name: nginx-azurefiles-pvc-acinode
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: azurefilespvc
      volumes:
      - name: azurefilespvc
        persistentVolumeClaim:
          claimName: azurefile
      nodeSelector:
        kubernetes.io/role: agent
        beta.kubernetes.io/os: linux
        type: virtual-kubelet
      tolerations:
      - key: virtual-kubelet.io/provider
        operator: Exists
      - key: azure.com/aci
        effect: NoSchedule
```

kubectl コマンドで AKS へ apply します。

```
kubectl apply -f dp-azurefiles-pvc-vol-acinode.yaml
```

作成後に、状態を確認すると、仮想ノードへ Pod はスケジューリングされていますが、STATUS が`ProviderFailed`となり Pod が起動していません。
また、`describe` コマンドで詳細を確認すると、

`Message:          pod nginx-azurefiles-pvc-acinode-7d55d7679b-ftb5n requires volume azurefilespvc which is of an unsupported type`
のメッセージが出ていました。

```
$ kubectl get po -o wide
NAME                                            READY   STATUS           RESTARTS   AGE   IP       NODE                     NOMINATED NODE   READINESS GATES
nginx-azurefiles-pvc-acinode-7d55d7679b-ftb5n   0/1     ProviderFailed   0          31s   <none>   virtual-node-aci-linux   <none>           <none>
nginx-azurefiles-pvc-acinode-7d55d7679b-kpwq4   0/1     ProviderFailed   0          31s   <none>   virtual-node-aci-linux   <none>           <none>
nginx-azurefiles-pvc-acinode-7d55d7679b-pbwbg   0/1     ProviderFailed   0          31s   <none>   virtual-node-aci-linux   <none>           <none>
nginx-azurefiles-pvc-acinode-7d55d7679b-ztm9g   0/1     ProviderFailed   0          31s   <none>   virtual-node-aci-linux   <none>           <none>
$ kubectl describe po nginx-azurefiles-pvc-acinode-7d55d7679b-ftb5n 
Name:             nginx-azurefiles-pvc-acinode-7d55d7679b-ftb5n
Namespace:        default
Priority:         0
Service Account:  default
Node:             virtual-node-aci-linux/
Labels:           app=nginx-azurefiles-pvc-acinode
                  node=aci
                  pod-template-hash=7d55d7679b
Annotations:      <none>
Status:           Pending
Reason:           ProviderFailed
Message:          pod nginx-azurefiles-pvc-acinode-7d55d7679b-ftb5n requires volume azurefilespvc which is of an unsupported type
IP:               
・・・ 省略 ・・・
Events:
  Type     Reason                Age                 From                                   Message
  ----     ------                ----                ----                                   -------
  Normal   Scheduled             98s                 default-scheduler                      Successfully assigned default/nginx-azurefiles-pvc-acinode-7d55d7679b-ftb5n to virtual-node-aci-linux
  Warning  ProviderCreateFailed  11s (x14 over 98s)  virtual-node-aci-linux/pod-controller  pod nginx-azurefiles-pvc-acinode-7d55d7679b-ftb5n requires volume azurefilespvc which is of an unsupported type
$
```
上記 Deployment から、仮想ノード用の設定である、`tolerations`と`nodeSelector`を外して、通常の AKS ノードへのデプロイを試したところ、問題なく起動し、ファイル共有も動作することが確認できます。

```
$ kubectl get po -o wide
NAME                                            READY   STATUS    RESTARTS   AGE   IP            NODE                                NOMINATED NODE   READINESS GATES
nginx-azurefiles-pvc-acinode-7cdf6c6c9f-6smcx   1/1     Running   0          76s   10.224.0.59   aks-agentpool-19611256-vmss000002   <none>           <none>
nginx-azurefiles-pvc-acinode-7cdf6c6c9f-9dj4j   1/1     Running   0          41s   10.224.0.11   aks-agentpool-19611256-vmss000002   <none>           <none>
nginx-azurefiles-pvc-acinode-7cdf6c6c9f-bhs6h   1/1     Running   0          41s   10.224.0.56   aks-agentpool-19611256-vmss000002   <none>           <none>
nginx-azurefiles-pvc-acinode-7cdf6c6c9f-zjvnn   1/1     Running   0          76s   10.224.0.39   aks-agentpool-19611256-vmss000002   <none>           <none>
```

:::note 
冒頭からの、繰り返しとなりますが、仮想ノードの制限事項にある通り
* AKS 仮想ノードは現在（本記事執筆時点）、__PV （ Persitent volume ）、PVC （ Persistent Volume Claim ）をサポートしていません__
* 仮想ノード上の Pod よりファイル共有を利用したい場合は、インラインボリュームとしてマウントします
* この制限事項は、CSIドライバーでも同様となります

__PV （ Persitent volume ）、PVC （ Persistent Volume Claim ）を利用できないのは、ハマりポイントだと思いますのでご注意ください！__
:::

## 4.3 Azure ファイル共有をインラインボリュームとしてマウントする
それでは、正規の方法で試してみます。
AKS 仮想ノードは現在（本記事執筆時点）、PV （ Persitent volume ）、PVC （ Persistent Volume Claim ）をサポートしていないため、ファイル共有をインラインボリュームとしてマウントしていきます。
さきほどの Deployment より、PV/PVC を利用する形から、[ファイル共有をインライン ボリュームとしてマウントする](https://learn.microsoft.com/ja-jp/azure/aks/azure-csi-files-storage-provision#mount-file-share-as-an-inline-volume) 方法で Pod から直接インラインマウントします。

https://learn.microsoft.com/ja-jp/azure/aks/azure-csi-files-storage-provision#create-an-azure-file-share

``` dp-csi-azurefiles-inline-vol-acinode.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-azurefiles-inline-vol-aci
  labels:
    node: aci
    app: nginx-azurefiles-inline-vol-aci
spec:
  strategy:
    type: RollingUpdate
  replicas: 4
  selector:
    matchLabels:
      node: aci
      app: nginx-azurefiles-inline-vol-aci
  template:
    metadata:
      name: nginx-azurefiles-inline-vol-aci
      labels:
        app: nginx-azurefiles-inline-vol-aci
        node: aci
    spec:
      containers:
      - image: docker.io/nginx:1.20
        name: nginx-azurefiles-inline-vol-aci
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: azurefile
# Mount directly from Pod using Secret via azureFile instead of pvc
      volumes:
      - name: azurefile
        csi:
          driver: file.csi.azure.com
          readOnly: false
          volumeAttributes:
            secretName: azure-secret  # required
            shareName: aksshare  # required
            mountOptions: "dir_mode=0777,file_mode=0777,cache=strict,actimeo=30,nosharesock"  # optional
      nodeSelector:
        kubernetes.io/role: agent
        beta.kubernetes.io/os: linux
        type: virtual-kubelet
      tolerations:
      - key: virtual-kubelet.io/provider
        operator: Exists
      - key: azure.com/aci
        effect: NoSchedule
```

kubectl コマンドで AKS へ apply します。

```
kubectl apply -f dp-csi-azurefiles-inline-vol-acinode.yaml
```

Pod の状態を確認すると、STATUS が `Running`となり、同一の Azure ファイル共有をマウントした複数の Pod を仮想ノード上で動作できることを確認できました。

```
$ kubectl get pod -o wide
NAME                                             READY   STATUS    RESTARTS   AGE    IP           NODE                     NOMINATED NODE   READINESS GATES
nginx-azurefiles-inline-vol-aci-f5d9bd68-fsmk6   1/1     Running   0          101s   10.239.0.7   virtual-node-aci-linux   <none>           <none>
nginx-azurefiles-inline-vol-aci-f5d9bd68-h6w2g   1/1     Running   0          101s   10.239.0.5   virtual-node-aci-linux   <none>           <none>
nginx-azurefiles-inline-vol-aci-f5d9bd68-n752t   1/1     Running   0          101s   10.239.0.4   virtual-node-aci-linux   <none>           <none>
nginx-azurefiles-inline-vol-aci-f5d9bd68-wc5nm   1/1     Running   0          101s   10.239.0.6   virtual-node-aci-linux   <none>           <none>
```

これで、ACI 仮想ノード上の複数 Podより、 ReadWriteMany 可能な Azure Files の共有フォルダ（ Azure ファイル共有）をマウントすることができました。
あとは、ストレージアカウント上の共有フォルダへ何かしら Web コンテンツ（ js や html、画像等）を格納し、以下のような Kubernetes Service （または Ingress ）を作成することで、外部よりアクセス可能になります。

``` svc.yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-azurefiles
  labels:
    node: aci
    app: nginx-azurefiles
spec:
  ports:
    - name: nginx-azurefiles
      port: 80
      targetPort: 80
  selector:
    node: aci
    app: nginx-azurefiles-inline-vol-aci
  type: LoadBalancer
```

# 5. まとめ
Azure ファイル共有を利用する場合、仮想ノード使用時の制限として

* __AKS 仮想ノードは現在（本記事執筆時点）、PV （ Persitent volume ）、PVC （ Persistent Volume Claim ）をサポートしていません。__
* AKS 仮想ノード上に作成された Pod で Azure ファイル共有を使用する必要がある場合は、__[ファイル共有をインライン ボリュームとしてマウントする](https://learn.microsoft.com/ja-jp/azure/aks/azure-csi-files-storage-provision#mount-file-share-as-an-inline-volume)必要があります。__
```
・・・ デプロイメントからの抜粋 ・・・
    spec:
      containers:
      - image: docker.io/nginx:1.20
        name: nginx-azurefiles-inline-vol-aci
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: azurefile
# pvc ではなく、Secret を使って Pod から azureFile を直接インラインマウント  
      volumes:
      - name: azurefile
        csi:
          driver: file.csi.azure.com
          readOnly: false
          volumeAttributes:
            secretName: azure-secret  # required
            shareName: aksshare  # required
            mountOptions: "dir_mode=0777,file_mode=0777,cache=strict,actimeo=30,nosharesock"  # optional
# 仮想ノード上のみに Pod デプロイを限定する場合に nodeSelector 設定（バースト時のみ仮想ノードへ Pod をスケジュールしたい場合は定義しない）  
```

また、Pod を仮想ノード上で動かすために

* 仮想ノードには [taints](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) が設定されているため、[toleration](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) をPodへ設定する必要があります。
* 上記 `tolerations` に加えて、Pod へ [nodeSelector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) を設定することで、仮想ノード上に限定して Pod を動かすことが可能です。
* 言い換えると、バースト時のみ、仮想ノードへ Pod をスケジューリングしたい場合は `nodeSelector`を定義しません。

```
・・・ デプロイメントからの抜粋 ・・・
    spec:
# 仮想ノード上のみに Pod デプロイを限定する場合に nodeSelector 設定（バースト時のみ仮想ノードへ Pod をスケジュールしたい場合はあえては nodeSelector を定義しない）  
      nodeSelector:
        kubernetes.io/role: agent
        beta.kubernetes.io/os: linux
        type: virtual-kubelet
# 仮想ノード上へ Pod をスケジュール可能なようにさせるために tolerations を設定
      tolerations:
      - key: virtual-kubelet.io/provider
        operator: Exists
      - key: azure.com/aci
        effect: NoSchedule
```

# 6. 参考
記事中でリンクしている物とほぼ同じですが

* https://docs.microsoft.com/ja-jp/azure/aks/virtual-nodes
* https://learn.microsoft.com/ja-jp/azure/aks/azure-csi-files-storage-provision
* https://docs.microsoft.com/ja-jp/azure/aks/virtual-nodes-portal#deploy-a-sample-app
* https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
* https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
* https://kubernetes.io/docs/concepts/storage/volumes/#azurefile
* https://kubernetes.io/docs/concepts/storage/volumes/#azurefile-csi-migration
* https://kubernetes.io/docs/concepts/storage/volumes/#azurefile-csi-migration-complete
* https://github.com/kubernetes/examples/blob/master/staging/volumes/azure_file/README.md

# 7. さいごに
最後まで、閲覧ありがとうございます。
今後 ACI 仮想ノード上の複数 Pod より、ReadWriteMany 可能な Azure Files の共有フォルダをマウントする際に、こちらのハマりポイントなど、お役に立てれば幸いです。
