---
title: "kubernetes の node のバージョンアップをダウンタイムなしで実行する"
date: 2018-02-15T15:16:31+09:00
draft: false
---

node pool の概要はこちらです。  
https://cloud.google.com/container-engine/docs/node-pools?hl=ja

今回は`1.6.4`から`1.6.6`にバージョンアップをします。

さて、kubernetes の node pool のバージョンアップをやるには  
1, 新しい node pool を作成する。  
2, 古い node に対して unschedule を設定する。  
3, pods を再構築して新しい node に乗るかを確認する。  
4, すべての pods を乗せ変える。  

#### 1, 新しい node pool を作成する。  
コマンドはこちらです。

    $ gcloud container node-pools create NODE_NAME --cluster=CLUSTER_NAME --machine-type=n1-standard-1 --disk-size=100 --num-nodes=2 --zone=asia-east1-a

zone に指定する asia-east1-a は cluster のzoneです。  
machine type とかは適宜変えましょう。  
node pool の一覧はこのようにして確かめれます。
```
$ gcloud container node-pools list --cluster=CLUSTER_NAME
NAME    MACHINE_TYPE   DISK_SIZE_GB  NODE_VERSION
pool1   n1-standard-1  100           1.6.4
pool2   n1-standard-1  100           1.6.6
```

#### 2, 古い node に対して unschedule を設定する。
node の情報を取得して、それぞれに対して unschedule を設定します。
```
$ kubectl get no
NAME                                   STATUS    AGE       VERSION
gke-hoge-cluster-pool1-00625cca-1j5h   Ready     2d        v1.6.4
gke-hoge-cluster-pool1-00625cca-d2fl   Ready     2d        v1.6.4
gke-hoge-cluster-pool1-00625cca-wuim   Ready     2d        v1.6.4
gke-hoge-cluster-pool2-f74157a4-glr2   Ready     3m        v1.6.6
gke-hoge-cluster-pool3-f74157a4-jbct   Ready     3m        v1.6.6
```

そしたら`gke-hoge-cluster-pool1-00625cca-wuim`に対して unschedule を設定しましょう。

```
$ kubectl cordon gke-hoge-cluster-pool1-00625cca-wuim
node "gke-hoge-cluster-pool1-00625cca-wuim" cordoned
```

古いすべての node に対して実行すると以下のようになります。
status が SchedulingDisabled になっています。
```
kubectl get node
NAME                                   STATUS                     AGE       VERSION
gke-hoge-cluster-pool1-00625cca-1j5h   Ready,SchedulingDisabled   2d        v1.6.4
gke-hoge-cluster-pool1-00625cca-d2fl   Ready,SchedulingDisabled   2d        v1.6.4
gke-hoge-cluster-pool1-00625cca-wuim   Ready,SchedulingDisabled   2d        v1.6.4
gke-hoge-cluster-pool2-f74157a4-glr2   Ready                      5m        v1.6.6
gke-hoge-cluster-pool2-f74157a4-jbct   Ready                      5m        v1.6.6
```

こうなることで、これから作られる pods はすべて `gke-hoge-cluster-pool2-f74157a4-glr2` か `gke-hoge-cluster-pool2-f74157a4-jbct` 上に作られることになります。

#### 3, pods を再構築して新しい node に乗るかを確認する。  

さて、今までの pods がどこに乗っているのかを確認しましょう。
```
$ kubectl get po -o wide -n hogehoge
NAME                     READY     STATUS    RESTARTS   AGE       IP           NODE
bra1-2107184181-vwt7p     2/2       Running   0          2d        10.84.1.13   gke-hoge-cluster-pool1-00625cca-d2fl
bra2-1613048047-5fw6x     1/1       Running   0          23m       10.84.1.35   gke-hoge-cluster-pool1-00625cca-wuim 
```

各nodeが古いもの（1.6.4）に乗っていることがわかります。
node　を新しくするには
```
$ kubectl delete po bra1-2107184181-vwt7p -n hogehoge
```
とするか、
```
$ kubectl scale deploy bra1 --replicas 0 -n hogehoge
# pods が消されたことを確認した後に
$ kubectl scale deploy bra1 --replicas 1 -n hogehoge
```
とすると

```
NAME                     READY     STATUS    RESTARTS   AGE       IP           NODE
bra1-2107184181-vwt7p     2/2       Running   0          2d        10.84.1.13   gke-hoge-cluster-pool2-f74157a4-glr2
bra2-1613048047-5fw6x     1/1       Running   0          23m       10.84.1.35   gke-hoge-cluster-pool2-f74157a4-jbct 
```

変更されていることがわかります。  
新しい　node にアプリケーションを移すと node 自体の ip アドレスが変わるので sql などの　whitelist に追加しましょう。
#### 4, すべての pods を乗せ変える。

あとは3の繰り返しです。 Yay !



#### 参考
[GKEのノードプールを利用したKubernetesのアップグレード](http://qiita.com/strsk/items/f548940a3fcac54e05ff)