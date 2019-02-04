+++
title = "第一次玩 operator-sdk 就上手"
date = 2019-02-04T14:33:14+08:00
tags = ["operator-sdk", "operator", "Kubernetes", "helm"]
categories = ["Kubernetes"]
+++

## 文章脈絡
- 前言
- Operator Framework 是什麼
- 認識 operator-sdk CLI
- 實作開始
    1. 建立 K8s 環境
    2. 安裝 operator-sdk CLI
    3. 用 operator-sdk CLI 建立一個 operator
    4. 了解一下新產生的 helm operator 中的檔案
    5. 編輯檔案
    6. 先在 K8s 中 deploy CRD
    7. Build operator container
    8. 在 K8s 中部署 operator
    9. 在 K8s 中部署自己定義的 custom resource
- 結語

## 前言

在接觸 Kubernetes 一陣子後，會發現一堆 operators 。  
而很殘念der，自己到目前都還沒有真正好好的玩過。  
雖然在 operator-sdk 出現以前，有很多人都自己手刻 operator，不過既然 operator 目前都有個 Framework 了，當然就來玩它囉！  

## Operator Framework 是什麼

- 是一個 open source toolkit
- 管理 Kubernetes native applications, called operators, in an effective, automated, and scalable way
- 這個 Framework 有兩個主要的專案：
    - [Operator SDK](https://github.com/operator-framework/operator-sdk): 就可以用它 build operator。
    - [Operator Lifecycle Manager](https://github.com/operator-framework/operator-lifecycle-manager) (OLM): 可以管理 operators 和 CRUD Kubernetes resource 用...。(可以到這邊玩玩：https://www.katacoda.com/openshift/courses/operatorframework/operator-lifecycle-manager ，簡單玩完後會發現功能蠻強大的)

## 認識 operator-sdk CLI

```
$ operator-sdk -h
An SDK for building operators with ease

Usage:
  operator-sdk [command]

Available Commands:
  add         Adds a controller or resource to the project
  build       Compiles code and builds artifacts
  completion  Generators for shell completions
  generate    Invokes specific generator
  help        Help about any command
  migrate     Adds source code to an operator
  new         Creates a new operator application
  olm-catalog Invokes a olm-catalog command
  print-deps  Print Golang packages and versions required to run the operator
  run         Runs a generic operator
  scorecard   Run scorecard tests
  test        Tests the operator
  up          Launches the operator

$ operator-sdk --version
operator-sdk version v0.4.0+git
```

本次會用到的指令就只有一個：`operator-sdk new` - 產生一個新的 operator 專案


## 實作開始

### 1. 建立 K8s 環境

Ref: https://github.com/operator-framework/operator-sdk#prerequisites

- 需要準備的環境有：
    - dep version v0.5.0+.
    - git
    - go version v1.10+.
    - docker version 17.03+.
    - kubectl version v1.11.0+.
    - Access to a kubernetes v.1.11.0+ cluster.
    - Helm & tiller server

參考安裝：

- [CentOS with kubeadm@1.12.4](https://github.com/sufuf3/play-k8s-operator/blob/master/vagrant/Vagrantfile): 以下範例都是使用這個 VM 來實作的
- [Ubuntu with kubeadm@1.12.4](https://github.com/sufuf3/hands-on-w-tutorials/tree/master/2019-01-03)

### 2. 安裝 operator-sdk CLI

執行方式如下：

```
go get -u github.com/golang/dep/cmd/dep
go get -u github.com/operator-framework/operator-sdk
cd ~/go/src/github.com/operator-framework/operator-sdk
make dep
make install
```

### 3. 用 operator-sdk CLI 建立一個 operator

執行方式如下：

```
mkdir -p $GOPATH/src/github.com/<GitHub_ID>
cd $GOPATH/src/github.com/<GitHub_ID>
operator-sdk new <GitHub_Repo_Name>
cd <GitHub_Repo_Name>/
```

因為之後開發其他專案的關係，本次使用 helm 來練習。產生新專案的方式如下：
`operator-sdk new <project-name> --type=helm --kind=<kind> --api-version=<group/version>`

* `--type=helm` 是來建立 Helm operator operator 用的。
* Required: `--kind` - 是我們想打算 CRD 要定義的 kind
* Required: `--api-version` - 是我們想要在這專案中的 CRD 想定義哪樣的 group/version。

如：

```
$ mkdir -p $GOPATH/src/github.com/sufuf3
$ cd $GOPATH/src/github.com/sufuf3
$ operator-sdk new play-k8s-operator --api-version=samina.fu.com/v1alpha1 --kind=MyTest --type helm
INFO[0000] Creating new Helm operator 'play-k8s-operator'.
INFO[0000] Create helm-charts/mytest/
INFO[0000] Create build/Dockerfile
INFO[0000] Create watches.yaml
INFO[0000] Create deploy/service_account.yaml
INFO[0000] Create deploy/role.yaml
INFO[0000] Create deploy/role_binding.yaml
INFO[0000] Create deploy/operator.yaml
INFO[0000] Create deploy/crds/samina_v1alpha1_mytest_crd.yaml
INFO[0000] Create deploy/crds/samina_v1alpha1_mytest_cr.yaml
INFO[0000] Run git init ...
Initialized empty Git repository in /home/vagrant/go/src/github.com/sufuf3/play-k8s-operator/.git/
INFO[0000] Run git init done
INFO[0000] Project creation complete.
```

跑完 `operator-sdk new <project-name> --type=helm --kind=<kind> --api-version=<group/version>`，進到資料夾中會發現它產生好的檔案，並且用 git log 會發現有一個 commit

```
$ cd play-k8s-operator/
$ $ tree
.
├── build
│   └── Dockerfile
├── deploy
│   ├── crds
│   │   ├── samina_v1alpha1_mytest_crd.yaml
│   │   └── samina_v1alpha1_mytest_cr.yaml
│   ├── operator.yaml
│   ├── role_binding.yaml
│   ├── role.yaml
│   └── service_account.yaml
├── helm-charts
│   └── mytest
│       ├── charts
│       ├── Chart.yaml
│       ├── templates
│       │   ├── deployment.yaml
│       │   ├── _helpers.tpl
│       │   ├── ingress.yaml
│       │   ├── NOTES.txt
│       │   ├── service.yaml
│       │   └── tests
│       │       └── test-connection.yaml
│       └── values.yaml
└── watches.yaml

$ git log
commit f0a174ce1deca8ef3b38c911b98b102ca7dfd27f
Author: Samina Fu <sufuf3@gmail.com>
Date:   Sat Feb 2 12:11:10 2019 +0000

    INITIAL COMMIT
```

### 4. 了解一下新產生的 helm operator 中的檔案

Ref: https://github.com/operator-framework/operator-sdk/blob/master/doc/helm/project_layout.md

從上面的檔案中會出現以下的資料夾和檔案：

- **`deploy`** 資料夾：包含一組可以 deploy 到 K8s cluser 上的 resource files
    - **`deploy/service_account.yaml`**: 建立一個這個專案名稱的 ServiceAccount
    - **`deploy/role.yaml`**: 建立一個 role 定義它能存取 K8s cluster 的規則
    - **`deploy/role_binding.yaml`**: 把上面的 ServiceAccount 和 Role bind 起來，也就是說，該 ServiceAccount 擁有這個 Role 定義的存取 rule
    - **`deploy/operator.yaml`**: 是 deploy operator 到 K8s cluster 上的 deployment，使用的 ServiceAccount 是 我們在 `deploy/service_account.yaml` 定義的名稱
    - **`deploy/crds/GROUP_VERSION_CRDNAME_crd.yaml`**: 就是我們自己定義的 resource type，More detail: [Extend the Kubernetes API with CustomResourceDefinitions](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/)
    - **`deploy/crds/GROUP_VERSION_CRDNAME_cr.yaml`**: 是我們要 deploy 客制的 resource，CR 代表 Custom Resource ，More detail: [Create custom objects](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#create-custom-objects)
而我們也不用擔心變成 K8s API 來呼叫會出現問題，因為這檔案中的 spec 就是完全複製 `helm-charts` 底下的 values.yaml，所以原本用 helm 怎麼定義 values.yaml 中的值，這份檔案就怎麼定義。反過來說，原本我們的 values.yaml 中的 key&value 怎麼寫的，在這邊就怎麼寫囉！
- **`helm-charts/<kind>`** 資料夾: 是熟悉的 helm-charts，用 `helm create` 指令會產生出來的檔案。
- **`build/Dockerfile`**: 是 operator 的 Dockerfile，之後當然要 docker build 起來
- **`watches.yaml`**: 內容包含 Group, Version, Kind, 和 Helm chart 路徑位置


### 5. 編輯檔案

#### 1. `watches.yaml`

default 的 operator 會依據 `watches.yaml` 這個檔案來監控 我們自己定義的 kind 這個 resource 的事件。基本上，在我們執行 `operator-sdk new` 就已經依據我們定義的名稱產生好，所以這份檔案基本上**不需要修改**。


```
---
- version: v1alpha1
  group: samina.fu.com
  kind: MyTest
  chart: /opt/helm/helm-charts/mytest
```

#### 2. helm-charts

依自己喜好來編輯自己的 helm-charts 囉！  
使用 `helm install --name NAME --dry-run --debug ./helm-charts/mytest/` 來 debug。  

參考：https://github.com/sufuf3/play-k8s-operator/blob/master/helm-dry-run.log  

### 6. 先在 K8s 中 deploy CRD

```
kubectl create -f deploy/crds/GROUP_VERSION_CRDNAME_crd.yaml
```

eg.

```
kubectl create -f deploy/crds/samina_v1alpha1_mytest_crd.yaml
```

然后一个新的 RESTful API 就創建了：

```
$ curl -s https://172.17.8.100:6443/apis/samina.fu.com/v1alpha1/namespaces/*/mytests -k | jq
{
  "apiVersion": "samina.fu.com/v1alpha1",
  "items": [],
  "kind": "MyTestList",
  "metadata": {
    "continue": "",
    "resourceVersion": "233252",
    "selfLink": "/apis/samina.fu.com/v1alpha1/namespaces/*/mytests"
  }
}
```

### 7. Build operator container

<!---#### 1. 修改 `build/Dockerfile`


如果沒有什麼特殊需求可以不用調整這邊個檔案。不過因為目前最新的 v0.4.0 沒有我需要的[功能](https://github.com/operator-framework/operator-sdk/pull/925) 所以以下會是重 build helm-operator image 並且修改 Dockerflie

1. Build helm-operator image
參考 https://github.com/operator-framework/operator-sdk/blob/master/.travis.yml 來 build image
```
cd ~/go/src/github.com/operator-framework/operator-sdk
make test/ci-helm
./hack/image/build-helm-image.sh sufuf3/helm-operator:latest
```
用 `docker images` 就會看到自己 build 的 image 了。再把它 push 到 docker hub 就大功告成。

2. 修改 build/Dockerfile
再次回到自己的專案， eg. `go/src/github.com/sufuf3/play-k8s-operator` 修改 build/Dockerfile
(記得自己的 username 要調整成自己的唷！)

```
FROM sufuf3/helm-operator:latest
#FROM quay.io/operator-framework/helm-operator:v0.4.0

COPY helm-charts/ ${HOME}/helm-charts/
COPY watches.yaml ${HOME}/watches.yaml
```

#### 2. Build 自己的 docker image--->
可以依據[官方](https://github.com/operator-framework/operator-sdk/blob/master/doc/helm/user-guide.md#1-run-as-a-pod-inside-a-kubernetes-cluster)提供的 Build 方式 `operator-sdk build registry_name/username/image_name:v0.0.1` + `docker push` 來進行。
不過 operator 維護上，最好專注在 operator 的定義，至於 build docker 這件事情可以交給懶人法 - dockerhub 自動 Build。

eg.

![](https://i.imgur.com/bNytZnq.png)


### 8. 在 K8s 中部署 operator 

#### 1. 修改 `deploy/operator.yaml` 的 image 來源

```
sed -i 's|REPLACE_IMAGE|repository/username/images_name:version|g' deploy/operator.yaml
```

eg.

```
sed -i 's|REPLACE_IMAGE|sufuf3/my-operator-test:latest|g'  deploy/operator.yaml
```

#### 2. 部署 operator

```
kubectl create -f deploy/service_account.yaml
kubectl create -f deploy/role.yaml
kubectl create -f deploy/role_binding.yaml
kubectl create -f deploy/operator.yaml
```

deploy 完後就可以看到我們的 operator 是 running 的  

```
$ kubectl get deploy,pod
NAME                                      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/play-k8s-operator   1         1         1            1           6m44s

NAME                                    READY   STATUS        RESTARTS   AGE
pod/play-k8s-operator-5c8fdc8fd-6m8tv   1/1     Running       0          6m44s
```

### 9. 在 K8s 中部署自己定義的 custom resource

編輯 `deploy/crds/GROUP_VERSION_CRDNAME_cr.yaml` 後 deploy。  
  
在我未來的專案設計流程中，會是需要先自己 create 一個 namespace 後，再依據已經建立的 namespace ，建立自己的 resource。  
不過即便是跑最新的 v0.4.0 或是用 master 上最新的 commit build 出 helm-operator image ，除了可以部署在 default namespace 外，目前都尚未成功可以部署在自己 create 的 namespace 中 。因此這部份待解決中...。  

#### 1. 編輯 `deploy/crds/GROUP_VERSION_CRDNAME_cr.yaml`

spec 底下就是自己定義的 values ，在我這個練習專案中，定義如下：

`vim deploy/crds/samina_v1alpha1_mytest_cr.yaml`

```
apiVersion: samina.fu.com/v1alpha1
kind: MyTest
metadata:
  name: example-mytest
spec:
  replicaCount: 1
  username: sufuf3
  containerName: mycontainer
  nameSpace: default
  image:
    repository: tensorflow/tensorflow
    tag: latest-jupyter
    pullPolicy: IfNotPresent
  nameOverride: ""
  fullnameOverride: ""
  service:
    type: NodePort
    port: 8888
    nodeport: 30566
  ingress:
    enabled: false
    annotations: {}
    paths: []
    hosts:
      - chart-example.local
    tls: []
  resources: {}
  nodeSelector: {}
  tolerations: []
  affinity: {}
```

#### 2. 建立自己的 resource

```
kubectl create -f deploy/crds/samina_v1alpha1_mytest_cr.yaml
```

結果：

```
$ kubectl get all -l app=mytest
NAME                                                           READY   STATUS    RESTARTS   AGE
pod/example-mytest-6bxnrell81eqsmdueqsqy7l7z-798dc58c8-g8q8q   1/1     Running   0          5m46s

NAME                                               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/example-mytest-6bxnrell81eqsmdueqsqy7l7z   NodePort   10.103.245.94   <none>        8888:30567/TCP   5m46s

NAME                                                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/example-mytest-6bxnrell81eqsmdueqsqy7l7z   1         1         1            1           5m46s

NAME                                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/example-mytest-6bxnrell81eqsmdueqsqy7l7z-798dc58c8   1         1         1       5m46s
```

透過 API:

```
$ curl -s https://172.17.8.100:6443/apis/samina.fu.com/v1alpha1/namespaces/default/mytests/example-mytest -k | jq
```

## 結語

本次的練習 GitHub repository 參考連結：https://github.com/sufuf3/play-k8s-operator  

這樣透過 operator-framework/operator-sdk 要完成自己的 CRD 和 operator 真的很快速與方便，可以想見，如果是維運人員，了解需求後，可以不用寫很多的 operator 程式，就可以快速部署 operator 真的蠻便利的。且後續一樣可以很方便的使用 K8s 的 API。根本就是懶人救星。  

不過，是否要使用 operator-framework/operator-sdk ，一切還是需要自行的評估是否合適自己，小心服用。  


## 參考連結
- https://github.com/operator-framework/operator-sdk/blob/master/doc/helm/user-guide.md
- https://github.com/operator-framework/operator-sdk/blob/master/doc/proposals/helm-operator.md
- https://github.com/operator-framework/getting-started
