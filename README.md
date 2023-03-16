# demo-aks-vnode-csi-azurefiles
---
---
title: AKS 仮想ノード( ACI ) 上の Pod から Azure ファイル共有をマウントする
tags: Azure AKS aci kubernetes virtual-kubelet
author: tbuchi888
slide: false
---

# 0. はじめに
Azure Kubernetes Service (以降 AKS ) は
* クラスター内の Worker node 数（ node pool ）を、需要に応じて、設定範囲内で自動で増減可能なオートスケーラー機能に加えて
* クラスターの許容量を超える急なバーストトラフィック等への対応として、OSS の Kubernetes kubelet の実装である、CNCF の [Virtual Kubelet](https://www.cncf.io/projects/virtual-kubelet/) を利用して、仮想ノードとして Azure Container Instances (以降 ACI )上に Pod をデプロイ可能です。

今回、ACI 仮想ノード（Virtual Node）上の複数 Pod より、ReadWriteMany 可能な Azure Files の共有フォルダ（ Azure ファイル共有）をマウントする方法について、以前書いたこちらの記事をベースに、CSI Storage Driver の内容へサンプルの Yaml などrewriteした記事となります。
なお、 仮想ノード（Virtual Node）の概念や利用方法等は、以前の記事の方が、詳しく書いていますので合わせて参考にしていただければ幸いです。

:::note info
Kubernetes バージョン 1.26 以降、in-tree persistent volume である kubernetes.io/azure-file は非推奨となりサポートされなくなくなるのに伴い、AKS の公式ドキュメント含めて、[CSI Storage Driver](https://learn.microsoft.com/ja-jp/azure/aks/csi-storage-drivers) を利用する形へシフトしていますので、そちらにに合わせて今回記事を rewrite しました
:::

:::note info
実行結果等を含めて本記事は、2023 年 3 月 14 日時点のものとなります
:::

# 1. 実行環境について
本記事は、以下 AKS バージョンで試しています。

```
$ kubectl get node
NAME                                STATUS   ROLES   AGE     VERSION
aks-agentpool-19611256-vmss000000   Ready    agent   5d23h   v1.24.9
virtual-node-aci-linux              Ready    agent   5d23h   v1.19.10-vk-azure-aci-1.4.8
```

# CSI Storage Driver 利用時の前回記事からの変更点（要点）
執筆時点でも、仮想ノード（Virtual Node）では、Persistent Volumes / Persitent Volume Claims はサポートされておらず、CSI Storage Driver を利用して PV/PVC をマウントしようとすると前回の記事と同様に以下のエラーにより、Podが起動されません。

```
$ kubectl get po
NAME                                            READY   STATUS           RESTARTS   AGE
nginx-azurefiles-pvc-acinode-7d55d7679b-b8k2h   0/1     ProviderFailed   0          26s
$ kubectl describe pod nginx-azurefiles-pvc-acinode-7d55d7679b-b8k2h
〜〜〜 中略 〜〜〜
Events:
  Type     Reason                Age                From                                   Message
  ----     ------                ----               ----                                   -------
  Normal   Scheduled             49s                default-scheduler                      Successfully assigned default/nginx-azurefiles-pvc-acinode-7d55d7679b-b8k2h to virtual-node-aci-linux
  Warning  ProviderCreateFailed  2s (x13 over 49s)  virtual-node-aci-linux/pod-controller  pod nginx-azurefiles-pvc-acinode-7d55d7679b-b8k2h requires volume azurefilespvc which is of an unsupported type
```

CSI Storage Driver 利用時も、仮想ノード（Virtual Node）上の複数 Pod より、ReadWriteMany 可能な Azure Files の共有フォルダ（Azure ファイル共有）をマウントする方法は、従来通り Inline volume によりマウントします。

https://learn.microsoft.com/ja-jp/azure/aks/azure-csi-files-storage-provision.md#mount-file-share-as-an-inline-volume

# 2. 事前準備
+ AKS で仮想ノードを有効化します。
    + 仮想ノードを有効化するには、AKS クラスタで`Azure CNI` ネットワーク プラグインを利用する必要があります。
    + 本記事では有効化手順の説明をしませんが、詳細は以下を参考にしてください。
    + [az コマンドでの CLI](https://docs.microsoft.com/ja-jp/azure/aks/virtual-nodes-cli)でも可能ですが、Azure Portalより [AKS クラスタ作成時に有効化する](https://docs.microsoft.com/ja-jp/azure/aks/virtual-nodes-portal)方法が楽だと思います。
+ 仮想ノード使用にあたって
    + 仮想ノード上へ Pod をデプロイするにはいくつか設定が必要（後述）
    + AKS での仮想ノード使用時の制限事項がいくつかある（後述）

# 3. 仮想ノード上へ Pod をデプロイするにはいくつか設定が必要
仮想ノードの[サンプルアプリ](https://docs.microsoft.com/ja-jp/azure/aks/virtual-nodes-portal#deploy-a-sample-app)を確認すると、[toleration](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) と[nodeSelector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) の定義がされていることが分かります。

* 仮想ノードへの Pod スケジューリングを許可するために、`tolerations`を設定する必要がある
    * 重要な Pod が意図せずに仮想ノードへスケジュールされないように 仮想ノードには[Taints](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)が設定されているため、`tolerations`により回避する設定を追加します。
* さらに、`nodeSelector`を設定することで、仮想ノード上に限定して Pod を動かすことが可能です。
    * なお、このサンプルは動作確認テストのために、`nodeSelector`を使用していますが、バースト時のみ、仮想ノードへ Pod をスケジューリングしたい場合は、`nodeSelector`を定義しません

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

# 4. Azure ファイル共有に関する、仮想ノード使用時の既知の制限事項
通常の AKS ノードと比べて、仮想ノードを利用する場合、いくつか制限事項があり、ファイル共有関連で以下の制限があったので、そちらを参考に試してみます。
その他、制限事項の詳細はこちらの[ドキュメント](https://docs.microsoft.com/ja-jp/azure/aks/virtual-nodes)を参照。

> Azure Files 共有のサポート[汎用 V2](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-account-overview#types-of-storage-accounts) と[汎用 V1](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-account-overview#types-of-storage-accounts) をマウントするボリューム。 [Azure Files 共有を含むボリューム](https://learn.microsoft.com/ja-jp/azure/aks/azure-files-csi)をマウントする手順に従います。

上記について、`Azure Files 共有を含むボリューム`に関するドキュメントのリンク先に誤りがあることと、こちらの表現だけでは分かりづらいのですが、実際には以下のような制限と対処方法が必要です。（公式ドキュメントへ修正版のPull Request送信中です）

> Volume mounting Azure Files share support [General-purpose V2](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-account-overview#types-of-storage-accounts) and [General-purpose V1](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-account-overview#types-of-storage-accounts). However virtual nodes currently don't support [Persistent Volumes](https://learn.microsoft.com/ja-jp/azure/aks/concepts-storage.md#persistent-volumes) and [Persistent Volume Claims](https://learn.microsoft.com/ja-jp/azure/aks/concepts-storage.md#persistent-volume-claims). Follow the instructions for mounting [a volume with Azure Files share as an inline volume](https://learn.microsoft.com/ja-jp/azure/aks/azure-csi-files-storage-provision.md#mount-file-share-as-an-inline-volume).

日本語では以下のようになるかと思います。
> Azure Files 共有の[汎用 V2](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-account-overview#types-of-storage-accounts) と[汎用 V1](https://learn.microsoft.com/ja-jp/azure/storage/common/storage-account-overview#types-of-storage-accounts) ボリュームマウントをサポートしています。しかしながら、仮想ノードでは [Persistens Volumes](https://learn.microsoft.com/ja-jp/azure/aks/concepts-storage.md#persistent-volumes) および [Persistent Volume Claim](https://learn.microsoft.com/ja-jp/azure/aks/concepts-storage.md#persistent-volume-claims) をサポートしていません。[Azure Files共有でボリュームをインラインボリュームとしてマウントする](https://learn.microsoft.com/ja-jp/azure/aks/azure-csi-files-storage-provision.md#mount-file-share-as-an-inline-volume))手順に従ってください。

## 4.1 Azure ファイル共有の事前準備
それでは、ファイル共有に関連する仮想ノードの制限事項（）、[Azure Files 共有を含むボリューム](https://docs.microsoft.com/ja-jp/azure/aks/azure-files-volume)に従ってファイル共有をマウントの準備をしていきます。

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

### 4.1.2. Kubernetes シークレットを作成する
    * Azure ファイル共有用へアクセスするために、Kubernetes の Secret を作成します。
    * 前の Step の実行結果である `AKS_PERS_STORAGE_ACCOUNT_NAME` と `STORAGE_KEY` 環境変数を利用していますが、 既存の Azure ストレージアカウントや、Portal か ら作成した場合には、ストレージアカウント名とアクセスキーを設定します。
  
```
kubectl create secret generic azure-secret --from-literal=azurestorageaccountname=$AKS_PERS_STORAGE_ACCOUNT_NAME --from-literal=azurestorageaccountkey=$STORAGE_KEY
```

以上で、ファイル共有を AKS で利用する準備が整いました。

## 4.2 ファイル共有を永続ボリューム（ Persitent volume ）としてマウントする
__結論から言うと、こちらの方法 ”ファイル共有を永続ボリュームとしてマウント” がハマりポイントでした。__

__通常の AKS ノード上ではうまくいくのですが、仮想ノード上ではうまくいきません。次のパートを試していただければと思います。__

ファイル共有を永続ボリューム（ Persitent volume ）としてマウントするために
前の Step で作成した、ファイル共有用の Secret を使って PV （ Persitent volume ）、PVC （ Persistent Volume Claim ）を静的に作成します。

``` pv-pvc.yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: azurefile
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  azureFile:
    secretName: azure-secret
    secretNamespace: default
    shareName: aksshare
    readOnly: false
  mountOptions:
  - dir_mode=0755
  - file_mode=0755
  - uid=1000
  - gid=1000
  - mfsymlinks
  - nobrl
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azurefile
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
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
NAME                         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
persistentvolume/azurefile   5Gi        RWX            Retain           Bound    default/azurefile                           5d1h

NAME                              STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/azurefile   Bound    azurefile   5Gi        RWX                           5d1h
```

次に、以下 Deployment を使って Pod を仮想ノードへデプロイします。

``` dp-azurefiles-pvc-vol-acinode.yml
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
kubectl apply -f dp-azurefiles-pvc-vol-acinode.yml
```

作成後に、状態を確認すると、仮想ノードへ Pod はスケジューリングされていますが、STATUS が`ProviderFailed`となり Pod が起動していません。
また、`describe` コマンドで詳細を確認すると、

`Message:        Pod nginx-azurefiles-aci-85c4d6b9fd-gr5tn requires volume azurefilespvc which is of an unsupported type`
のメッセージが出ていました。

```
$ kubectl get po -o wide
NAME                                    READY   STATUS           RESTARTS   AGE     IP             NODE                                NOMINATED NODE   READINESS GATES
nginx-azurefiles-aci-85c4d6b9fd-gr5tn   0/1     ProviderFailed   0          4h8m    <none>         virtual-node-aci-linux              <none>           <none>
nginx-azurefiles-aci-85c4d6b9fd-hpt9k   0/1     ProviderFailed   0          4h8m    <none>         virtual-node-aci-linux              <none>           <none>
nginx-azurefiles-aci-85c4d6b9fd-s6p8p   0/1     ProviderFailed   0          4h8m    <none>         virtual-node-aci-linux              <none>           <none>
nginx-azurefiles-aci-85c4d6b9fd-v6qvv   0/1     ProviderFailed   0          4h8m    <none>         virtual-node-aci-linux              <none>           <none>
$ kubectl describe po nginx-azurefiles-aci-85c4d6b9fd-gr5tn
Name:           nginx-azurefiles-aci-85c4d6b9fd-gr5tn
Namespace:      default
Priority:       0
Node:           virtual-node-aci-linux/
Labels:         app=nginx-azurefiles-aci
                node=aci
                pod-template-hash=85c4d6b9fd
Annotations:    <none>
Status:         Pending
Reason:         ProviderFailed
Message:        Pod nginx-azurefiles-aci-85c4d6b9fd-gr5tn requires volume azurefilespvc which is of an unsupported type
IP:
・・・ 省略 ・・・
Events:          <none>
    SecretName:  default-token-9fbh4    
$
```

問題、切り分けのため、上記 Deployment から、仮想ノード用の設定である、`tolerations`と`nodeSelector`を外して、通常の AKS ノードへのデプロイを試したところ、
問題なく起動し、ファイル共有も動作することが確認できたため、ドキュメントフィードバック機能を利用して内容を確認したところ、、、以下が分かりました。

* AKS 仮想ノードは現在（本記事執筆時点）、__永続ボリューム（Persistent Volume）をサポートしていない__
* 仮想ノード上の Pod よりファイル共有を利用したい場合は、インラインボリュームとしてマウントする

__Persistent Volume を利用できないのは、ハマりポイントでした。。。__

## 4.3 Azure ファイル共有をインラインボリュームとしてマウントする
__結論から言うと、仮想ノード上では、こちらの方法が正解でした。__

AKS 仮想ノードは現在（本記事執筆時点）、永続ボリューム（Persistent Volume）をサポートしていないため、ファイル共有をインラインボリュームとしてマウントしていきます。
さきほどの Deployment より、PV/PVC を利用する形から、[azureFile](https://kubernetes.io/docs/concepts/storage/volumes/#azurefile) で Pod から直接インラインマウントする形へ変更します。

``` dp-azurefiles-inline-vol-acinode.yml
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
        azureFile:
          secretName: azure-secret
          shareName: aksshare
          readOnly: false
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
kubectl apply -f dp-azurefiles-inline-vol-acinode.yml
```

Pod の状態を確認すると、STATUS が `Running`となり、同一の Azure ファイル共有をマウントした複数の Pod を仮想ノード上で動作できることを確認できました。

```
$ kubectl get pod -o wide
NAME                                                   READY   STATUS    RESTARTS   AGE     IP             NODE                                NOMINATED NODE   READINESS GATES
nginx-azurefiles-inline-vol-aci-589575b559-4jnd5       1/1     Running   0          4m1s    10.241.0.4     virtual-node-aci-linux              <none>           <none>
nginx-azurefiles-inline-vol-aci-589575b559-8nmvm       1/1     Running   0          3m24s   10.241.0.7     virtual-node-aci-linux              <none>           <none>
nginx-azurefiles-inline-vol-aci-589575b559-9kkd6       1/1     Running   0          3m24s   10.241.0.6     virtual-node-aci-linux              <none>           <none>
nginx-azurefiles-inline-vol-aci-589575b559-drdtr       1/1     Running   0          4m1s    10.241.0.5     virtual-node-aci-linux              <none>           <none>
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

* __AKS 仮想ノードは現在（本記事執筆時点）、永続ボリューム（Persistent Volume）をサポートしていません。__
* AKS 仮想ノード上に作成された Pod で Azure ファイル共有を使用する必要がある場合は、__[ファイル共有をインラインボリュームとしてマウントする](https://docs.microsoft.com/ja-jp/azure/aks/azure-files-volume#mount-file-share-as-an-inline-volume)必要があります。__

Pod を仮想ノード上で動かすために

* 仮想ノードには [taints](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) が設定されているため、[toleration](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) をPodへ設定する必要があります。
* 上記 `tolerations` に加えて、Pod へ [nodeSelector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) を設定することで、仮想ノード上に限定して Pod を動かすことが可能です。
* 言い換えると、バースト時のみ、仮想ノードへ Pod をスケジューリングしたい場合は `nodeSelector`を定義しません。

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
# pvc ではなく、azureFile により、Secret を使って Pod から直接マウント         
      volumes:
      - name: azurefile
        azureFile:
          secretName: azure-secret
          shareName: aksshare
          readOnly: false
# 仮想ノード上のみに Pod デプロイを限定する場合に nodeSelector 設定（バースト時のみ仮想ノードへ Pod をスケジュールしたい場合は定義しない）  
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

なお、本内容については、ドキュメントへ[フィードバック中](https://github.com/MicrosoftDocs/azure-docs/issues/84855)です。

# 6. 参考
記事中でリンクしている物とほぼ同じですが

* https://docs.microsoft.com/ja-jp/azure/aks/virtual-nodes
* https://docs.microsoft.com/ja-jp/azure/aks/azure-files-volume
* https://docs.microsoft.com/ja-jp/azure/aks/virtual-nodes-portal#deploy-a-sample-app
* https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
* https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/
* https://kubernetes.io/docs/concepts/storage/volumes/#azurefile
* https://github.com/kubernetes/examples/blob/master/staging/volumes/azure_file/README.md

# 7. さいごに
最後まで、閲覧ありがとうございます。
今後 ACI 仮想ノード上の複数 Pod より、ReadWriteMany 可能な Azure Files の共有フォルダをマウントする際に、こちらのハマりポイントなど、お役に立てれば幸いです。
