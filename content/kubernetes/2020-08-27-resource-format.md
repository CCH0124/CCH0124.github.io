---
title: kubernetes - day06
date: 2020-08-27
description: "資源定義與使用"
tags: [kubernetes]
draft: false
---

## 資源配置清單

`Kubernetes` 中資源基本上都是需要五個字段定義資源，分別是 `apiVersion`、`kind`、`metadata`、`spec` 和 `status`。以下分別介紹

- `apiVersion`
    - group/version
    - 定義 api-versions
- `kind`
    - 資源類型
- `metadata` 
    - 用來定義一些訊息，如名稱、所屬 `namespace` 與標籤等
- `spec`
    - 定義所期望的狀態
-  `status`
    - 紀錄當前對象狀態
    - 由 Kubernetes 維護，因此客戶端不需要去定義


這邊使用 `get` 方式取得 `kube-system` 的預設 `namespcae` 資訊，並以 `yaml` 輸出。

```shell
$ kubectl get namespace kube-system -o yaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: "2020-08-22T02:00:45Z"
  name: kube-system
  resourceVersion: "4"
  selfLink: /api/v1/namespaces/kube-system
  uid: 73058111-fb1f-48a6-b283-338207167625
spec:
  finalizers:
  - kubernetes
status:
  phase: Active
```

大致上相關資源都是以上面格式進行清單定義創建資源。以下練習建立一個 `namespace` 資源。為何不用定義跟上述一樣的字段 ? 在 `Kubernetes` 中，有些字段必須要填寫，有些則不用，而不用的 `Kubernetes` 會自動將期補齊。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: test
```

```shell
$ kubectl creat -f namespace-example.yaml
$ kubectl get namespace # 這邊會看到所建立的 test namespace 資源
```

```shell
# 自動補齊一些字段值
$ kubectl get namespace test -o yaml
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Namespace","metadata":{"annotations":{},"name":"test"}}
  creationTimestamp: "2020-08-22T04:24:30Z"
  name: test
  resourceVersion: "370645"
  selfLink: /api/v1/namespaces/test
  uid: bcfe414a-cab9-400c-b004-38672c0bc065
spec:
  finalizers:
  - kubernetes
status:
  phase: Active
```

雖然上面建立了 `namespace` 資源，但那些字段我們可以去使用 ? 在 `Kubernetes` 中有文檔可以翻閱查看，也可以使用 `kubectl explain` 的方式去查找，相同的會給一些相關資源必須的字段，和描述當中還附著官方文檔的鏈接，相當的好用。

### 總結

資源清單方式是建立資源的一種方式，當然也可以用指令去建立，下一個段落會說明如何去管理這些資源。但是資源清單有許多優勢，像是可配置屬性較多因此可以有較高層級的操作、可用於 `git` 版控追蹤等。

## 資源對象管理方式

在 `kubectl` 中有三種方式管理資源
- Imperative commands
- Imperative object configurations
- Declarative object configurations


### Imperative commands

- 透過指令方式管理資源
- 通常於開發或測試使用

```shell=
$ kubectl run/create/delete/label/scal/patch/edit ...
```

但在真實部署中會把上述的動作描寫成一個檔案，紀錄這些動作。然而這些指令很檢簡易的使用，缺點是可能透過這些指令寫成 `Shell` 而不好維護。當中 `edit` 指令可用於修改 `POD` 運行資源的檔案需求內容，但 `edit` 在實際部署時不適合，因為無法記錄修改了什麼。


### Imperative object

透過物件方式去描述增減刪資源等。會透過 `yaml` 或 `json` 管理資源。

建立

```shell=
$ kubectl create -f FILE.yaml
```

刪除

```shell=
$ kubectl delete -f FILE.yaml
```

`replace`，根據修改的檔案內容進行替換。但，自定義的 `yaml` 跟系統上面的 `yaml` 不一樣，因此無法覆蓋。少資源覆蓋多資源是不可行，應該是要一模一樣才對，透過 `-o yaml` 可以取得完整 `yaml`。在進行 `replace` 時，無定義的 `yaml` 都會被檢視出來，因此要先取得完整 `yaml` 在進行 `replace`。

```shell=
$ kubectl replace -f FILE.yaml
```

而利用這些資源管理，對於更新是要取得最新資源的檔案。並透過上述提到的指令進行相對應操作。


- [imperative-command](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/imperative-command/)



### Declarative object

同樣也是使用檔案描述，底層由 `kubernetes` 維護，也就是其他未定義欄位。因此會針對先前修改過的欄位比較，並進行刪或更新動作。相較於 `create` 是去創建，而 `apply` 則是維護。

```shell=
$ kubectl apply -f file # 可以遞規當前目錄下所有檔案和目錄
$ kubectl diff ... # 比較差異
```


### 總結

![](http://assets.digitalocean.com/articles/comics/imperative-declarative-k8s.jpg)
上圖為 digitalocean 提供。

透過 `create` 再 `replace` 操作，叫做*命令式配置*(Imperative object configurations)操作。告訴機器先做什麼，再做什麼，機器按照一條一條的命令規則去實作。

透過 `apply` 進行*創建*和*滾動*更新，叫做*聲明式 API*操作。告訴機器去做什麼，但不細說具體指令(`kubernetes` 會維護)，它會自動識別完成任務。

`replace` 的執行過程，是使用新的 `YAML` 文件中的 `API` 對象，替換原有的 `API` 對象。`API Server` 在響應命令式請求(Imperative commands)時，一次只能處理一個寫請求，否則就會產生衝突的可能，一條一條按順序執行響應。

`apply` 是執行了一個對原有 `API` 對象的 `Patch` 操作。類似還要 `set image` 和 `edit`。 `API Server` 在響應聲明式請求(Declarative object configurations)時，一次能處理多個寫操作，並且具備整合的能力。

## 參考資源
- [官方 物件管理](https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/)
- [Declarative vs imperative in Kubernetes](https://containers.goffinet.org/k8s/declarative.html)