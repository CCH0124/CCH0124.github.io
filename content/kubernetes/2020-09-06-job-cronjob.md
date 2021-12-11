---
title: kubernetes - day16
date: 2020-09-06
description: "Job 和 CronJob 控制器"
tags: [kubernetes]
draft: false
---

`Job` 控制器是一次性任務的 `POD` 控制器，期會創建一至多個 `POD`，只要容器運行過程正常的運行即不會有重啟的動作，否則需要依照重啟策略去運行，如果遇到節點故障未完成任務的 `POD` 則會被重新的調度。`Job` 完成任務的定義是 `Job` 控制器會記錄 `POD` 成功執行完任務的個數，並達到成功次數的值。運行 `Job` 方式可以是是否要以平行的方式去運行多個 `POD`。`CronJob` 可以想成是 `crontab`，用來管理 `Job` 控制器要運行的時間，也就是在未來的某一個時間上運行或是固定某一時段運行。前面文章所提到的控制器會比較希望應用程式是以一個 `Daemon` 來運行，而 `Job` 類型則是會以 `Task` 為主的應用程式。

- 非平行化
    - 一次性
    - 一次執行一個 `POD` 作業，直到達到定義成功的次數
- 平行化
    - 平行處理
    - 會設置工作列隊的數量，該數量可讓多個 `POD` 同時作業

## 建立 Job

`Job` 需要定義必要的 `template`，可透過 `kubectl explain job.spec` 去查看，至於*標籤選擇器*則會自動藉由 `template` 去創建。以下我們拿官網的範例實驗

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
#   backoffLimit: 4
```

>`Job` 建立 `POD` 無法使用 `restartPolicy` 的默認值(always)，因為 `Job` 並非要無限期運作。需要設置 `Never` 或 `OnFailure` 防止容器完成任務後重新啟動


同樣的使用 `apply` 進行部署。用 `get` 取得 `Job` 的資源，因為還在運行所以  `COMPLETIONS` 還是 0。
```shell
$ kubectl get jobs pi
NAME   COMPLETIONS   DURATION   AGE
pi     0/1           25s        25s
```
列出 `Job` 定義的 `POD`，其 `STATUS` 表示完成了任務。`NAME` 是以 `Job` 控制器名稱開頭，與前面介紹的控制器相同。
```shell
$ kubectl get pods -l job-name=pi
NAME       READY   STATUS      RESTARTS   AGE
pi-r29jg   0/1     Completed   0          75s
```

使用 `describe` 詳細查看該 `Job` 資源，觀察到 `Selector` 和 `Labels` 有值，其值是自動生成的，這邊不深入探討。

```shell
$ kubectl describe jobs pi
Name:           pi
Namespace:      default
Selector:       controller-uid=3a72f47b-8d1b-4771-aab6-12dd97b286dc
Labels:         controller-uid=3a72f47b-8d1b-4771-aab6-12dd97b286dc
                job-name=pi
Annotations:    <none>
Parallelism:    1
Completions:    1
Start Time:     Sat, 12 Sep 2020 08:22:15 +0000
Completed At:   Sat, 12 Sep 2020 08:22:46 +0000
Duration:       31s
Pods Statuses:  0 Running / 1 Succeeded / 0 Failed
Pod Template:
  Labels:  controller-uid=3a72f47b-8d1b-4771-aab6-12dd97b286dc
           job-name=pi
  Containers:
   pi:
    Image:      perl
    Port:       <none>
    Host Port:  <none>
    Command:
      perl
      -Mbignum=bpi
      -wle
      print bpi(2000)
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  103s  job-controller  Created pod: pi-r29jg
```

## 非平行式 Job

須將 `spec` 中 `parallelism` 字段設置為 1 (預設也是)，同時也設置 `completion` 表示期望成功的數量。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-1
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  completions: 5
```

使用 `-w` 觀察，`job-name` 會以 `Job` 資源清單定義的名稱做為值。

```shell
$ kubectl get pods -l job-name=pi-1 -w
NAME         READY   STATUS    RESTARTS   AGE
pi-1-dd7dd   1/1     Running   0          8s
pi-1-dd7dd   0/1     Completed   0          8s
pi-1-krfkf   0/1     Pending     0          0s
pi-1-krfkf   0/1     Pending     0          0s
pi-1-krfkf   0/1     ContainerCreating   0          0s
pi-1-krfkf   1/1     Running             0          2s
pi-1-krfkf   0/1     Completed           0          8s
pi-1-rw5b7   0/1     Pending             0          0s
pi-1-rw5b7   0/1     Pending             0          0s
pi-1-rw5b7   0/1     ContainerCreating   0          0s
pi-1-rw5b7   1/1     Running             0          2s
pi-1-rw5b7   0/1     Completed           0          8s
pi-1-9sxpn   0/1     Pending             0          0s
pi-1-9sxpn   0/1     Pending             0          0s
pi-1-9sxpn   0/1     ContainerCreating   0          0s
pi-1-9sxpn   1/1     Running             0          2s
pi-1-9sxpn   0/1     Completed           0          7s
pi-1-gphql   0/1     Pending             0          0s
pi-1-gphql   0/1     Pending             0          0s
pi-1-gphql   0/1     ContainerCreating   0          0s
pi-1-gphql   1/1     Running             0          2s
pi-1-gphql   0/1     Completed           0          9s
```

```shell
$ kubectl get jobs pi-1 -o wide # 完成 5 個任務
NAME   COMPLETIONS   DURATION   AGE     CONTAINERS   IMAGES   SELECTOR
pi-1   5/5           40s        4m31s   pi           perl     controller-uid=73e23e4f-ae77-404b-b5f0-6ac7306542a8 
$ kubectl get pods -l job-name=pi-1 -o wide
NAME         READY   STATUS      RESTARTS   AGE     IP          NODE                                       NOMINATED NODE   READINESS GATES
pi-1-9sxpn   0/1     Completed   0          6m11s   10.4.3.9    gke-cluster-1-default-pool-7dc8b11b-8kvr   <none>           <none>
pi-1-dd7dd   0/1     Completed   0          6m35s   10.4.3.6    gke-cluster-1-default-pool-7dc8b11b-8kvr   <none>           <none>
pi-1-gphql   0/1     Completed   0          6m4s    10.4.3.10   gke-cluster-1-default-pool-7dc8b11b-8kvr   <none>           <none>
pi-1-krfkf   0/1     Completed   0          6m27s   10.4.3.7    gke-cluster-1-default-pool-7dc8b11b-8kvr   <none>           <none>
pi-1-rw5b7   0/1     Completed   0          6m19s   10.4.3.8    gke-cluster-1-default-pool-7dc8b11b-8kvr   <none>           <none>
```

## 平行式

`parallelism` 字段設置為 1 以上的值即可。如果有順序性的話這種方式可能就不適合了。
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-2
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  completions: 7
  parallelism: 3
```
同樣的可以用以下方式去觀察
```shell
$ kubectl get pods -l job-name=pi-2 -w
$ kubectl get pods -l job-name=pi-2 -o wide -w
$ kubectl get jobs pi-2
NAME   COMPLETIONS   DURATION   AGE
pi-2   7/7           45s        4m10s
```

## Job 刪除
只要 `Job` 完成任務後，其 `POD` 即可刪除，就算不刪除也不占資源。假設 `Job` 任務沒順利完成，且不斷的在錯誤中循環，這將是一個麻煩，所幸的是 `K8s` 有給 `Job` 一些屬性去防止發生錯誤循環這件事

- `spec` 中 `activeDeadlineSeconds` 屬性
    - 活動的最長時間，超出定義值則終止
- `spec` 中 `backoffLimit`
    - 重啟次數定義，預設是 `6`

刪除 `Job` 資源
```shell
$ kubectl delete jobs [JOB_NAME]
$ kubectl delete -f JOB_RESOURCE.yaml
```

`Job` 這邊最後實驗剛介紹的兩個參數

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  completions: 10
  parallelism: 3
  activeDeadlineSeconds: 5
  backoffLimit: 3
```

透過以下結果發現全部任務失敗，其原因是我們用 `activeDeadlineSeconds` 和 `backoffLimit` 去限制運行時的資源。從 `describe` 的 `EVENTS` 來看有 `Job was active longer than specified deadline` 這個訊息。這邊要注意的是是 `activeDeadlineSeconds` 讓這任務失敗，非 `backoffLimit` 優先權來說是前者優先。

```shell
$ kubectl get pods -l job-name=pi -w
NAME       READY   STATUS        RESTARTS   AGE
pi-2bfdl   0/1     Terminating   0          18s
pi-wss7c   1/1     Terminating   0          18s
pi-zk7j8   1/1     Terminating   0          18s
pi-zk7j8   0/1     Terminating   0          19s
pi-wss7c   0/1     Terminating   0          19s
pi-wss7c   0/1     Terminating   0          19s
pi-2bfdl   0/1     Terminating   0          22s
pi-2bfdl   0/1     Terminating   0          22s
pi-wss7c   0/1     Terminating   0          22s
pi-wss7c   0/1     Terminating   0          22s
pi-zk7j8   0/1     Terminating   0          22s
pi-zk7j8   0/1     Terminating   0          22s
$ kubectl get jobs pi
NAME   COMPLETIONS   DURATION   AGE
pi     0/10          4m46s      4m46s
$ kubectl describe job.batch pi
Events:
  Type     Reason            Age    From            Message
  ----     ------            ----   ----            -------
  Normal   SuccessfulCreate  7m21s  job-controller  Created pod: pi-zk7j8
  Normal   SuccessfulCreate  7m21s  job-controller  Created pod: pi-2bfdl
  Normal   SuccessfulCreate  7m21s  job-controller  Created pod: pi-wss7c
  Normal   SuccessfulDelete  7m16s  job-controller  Deleted pod: pi-2bfdl
  Normal   SuccessfulDelete  7m16s  job-controller  Deleted pod: pi-wss7c
  Normal   SuccessfulDelete  7m16s  job-controller  Deleted pod: pi-zk7j8
  Warning  DeadlineExceeded  7m16s  job-controller  Job was active longer than specified deadline
```

## CronJob 創建
下面 `yaml` 執行每一分鐘運行一次運算 `pi` 的 `Job`
```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: pi
spec:
  schedule: "*/1 * * * *" # 調度時間
  jobTemplate: # 生成 Job 模板
    spec:
      template:
        spec:
          containers:
          - name: pi
            image: perl
            command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
          restartPolicy: Never
      completions: 10
      backoffLimit: 3
```

其概念像是 `Deployment` 下面有 `ReplicaSet` 管理 `POD`，這邊則是 `Job` 管理 `POD`。

```shell
$ kubectl get cronjob
NAME   SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
pi     */1 * * * *   False     0        <none>          13s
$ kubectl get jobs
NAME            COMPLETIONS   DURATION   AGE
pi-1599902940   0/10          4s         4s
$ kubectl get jobs # 每過一分鐘就執行
NAME            COMPLETIONS   DURATION   AGE
pi-1599902940   10/10         87s        3m26s
pi-1599903000   10/10         102s       2m26s
pi-1599903060   7/10          86s        86s
pi-1599903120   1/10          26s        26s
$ kubectl delete cronjob pi # 刪除 cronjob 資源
```

## 參考資源

- [官方 Job](https://kubernetes.io/zh/docs/concepts/workloads/controllers/job/)
- [官方 CronJob](https://kubernetes.io/zh/docs/concepts/workloads/controllers/cron-jobs/)