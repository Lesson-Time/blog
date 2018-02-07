---
title: "Kubernetes で ingress controller を使って https (TLS) 接続をするチュートリアル"
date: 2018-02-07T18:57:41+09:00
draft: false
---
弊社ではKubernetesでDocker Containerを管理しています。  
[レッスンタイム](https://lesson-time.com/)もそうですが、常時HTTPS接続をするために内部にnginx
proxy の役割をするServiceを使っていました。

ingress を使うと何がメリットかというと  
 - 一つのingressで複数Serviceにまたがった設定ができる。  
 - Serviceとして管理しないので分けて考えれる  
 - Google 推奨の構成  
といったところですね。  


このチュートリアルでは  

1, ingressを使ってアプリケーションを公開する  
2, TLSの設定をして、https接続する。  
の二つを行いたいと思います。


###  1, ingressを使ってアプリケーションを公開する  
##### 1.1, サービスを立ち上げる
まずはingressを使わないでサービスを立ち上げれるかチェックします。  
[こちらの公式ページ](https://kubernetes.io/docs/concepts/services-networking/connect-applications-service/)がとてもわかりやすいのですが、Deployment から exposeして外部に公開してもいいし、Serviceを記述して公開してもいいです。

例えばこういうDeploymentがあるとします。  
sandboxというnamespaceに一つアプリケーションを8080で公開する記述がされています。  
Rails だと3000ですね。
```
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: test-app
  namespace: sandbox
spec:
  replicas: 1
  revisionHistoryLimit: 3
  template:
    metadata:
      namespace: sandbox
      labels:
        run: test-app
        version: latest
    spec:
      containers:
      - name: test-app
        image: asia.gcr.io/project-id/image_name:latest
        ports:
        - containerPort: 8080
```

Terminal で
```
$ kubectl apply -f /path/to/deployment.yaml -n sandbox
```
とすると
```
$ kubectl get deploy,po,svc -n sandbox -o wide
xtra241-imac:~ yoshi$ kubectl get deploy,po,svc -n sandbox -o wide
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINER(S)   IMAGE(S)                                    SELECTOR
deploy/test-app      1         1         1            1           1h        test-app       asia.gcr.io/project-id/image_name:latest    run=test-app,version=latest

NAME                              READY     STATUS    RESTARTS   AGE       IP           NODE
po/test-app-1139287214-gwb5l      1/1       Running   0          1h        10.84.5.78   gke-cluster-pool-f74157a4-jbct

```
という感じでDeploymentとpodが生成されます。  
以下のコマンドでexpose して、serviceを生成します。
```
$ kubectl expose deploy/test-app -n sandbox --type=NodePort
```
External ip が `<nodes>` になっていたら完了です。  
```
$ kubectl get svc -n sandbox
NAME                  CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE       SELECTOR
svc/test-app      10.87.241.72   <nodes>       8080:30932/TCP   1h        run=test-app
```

これは以下のServiceと同意です。
```
apiVersion: v1
kind: Service
metadata:
  name: test-app
  namespace: sandbox
  labels:
    run: test-app
spec:
  type: NodePort
  selector:
    run: test-app
  ports:
  - port: 8080
    targetPort: 8080
    protocol: TCP
```


今の時点で、Deployment, Pods, Serviceが生成されました。  
NodePortで公開しているので、NodeにServiceが割り当てられ、IngressがService経由でPodsにアクセスすることができる状態になっています。


##### 1.2, Ingressの設定
Ingressは公式ドキュメントにしたがって記述するとすぐに設定できます。
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  namespace: sandbox
spec:
  backend:
    serviceName: test-app
    servicePort: 8080
```

これをk8s上にあげます。
```
$ kubectl apply -f /path/to/ingress.yaml -n sandbox
```

```
$ kubectl get ing -n sandbox
NAME               HOSTS     ADDRESS         PORTS     AGE
ing/test-ingress   *         35.227.212.97   80, 443   3h
```
ingress が正しく作成されているのが確認されました。  
Addressは設定されるまで少し時間がかかります。

```
$ kubectl describe int/test-ingress -n sandbox
Name:			test-ingress
Namespace:		sandbox
Address:		35.xxx.xxx.xx
Default backend:	test-app:8080 (10.84.5.78:8080)
TLS:
  test-app.com-ssl terminates
Rules:
  Host	Path	Backends
  ----	----	--------
  *	* 	test-app:8080 (10.84.5.78:8080)
Annotations:
  target-proxy:			k8s-tp-sandbox-lt-ingress--d27c22d5c9cb1c59
  forwarding-rule:		k8s-fw-sandbox-lt-ingress--d27c22d5c9cb1c59
  https-forwarding-rule:	k8s-fws-sandbox-lt-ingress--d27c22d5c9cb1c59
  https-target-proxy:		k8s-tps-sandbox-lt-ingress--d27c22d5c9cb1c59
  ssl-cert:			k8s-ssl-sandbox-lt-ingress--d27c22d5c9cb1c59
  static-ip:			k8s-fw-sandbox-lt-ingress--d27c22d5c9cb1c59
  url-map:			k8s-um-sandbox-lt-ingress--d27c22d5c9cb1c59
  backends:			{"k8s-be-30932--d27c22d5c9cb1c59":"HEALTHY"}
Events:
  FirstSeen	LastSeen	Count	From			SubObjectPath	Type		Reason	Message
  ---------	--------	-----	----			-------------	--------	------	-------
  3h		7m		27	loadbalancer-controller			Normal		Service	default backend set to test-app:30932

```

ここで、roles > backend が test-app:8080 になっているのを確認します。  
ブラウザで `35.xxx.xxx.xx` にアクセスすると正しく表示されていると思います。


#### 2, TLSの設定をして、https接続する。
これまで、 ingress > service への接続ができました。  
図示すると、
```
# 1.1
    internet
        |
  ------------
  [ Services ]
```
から
```
# 1.2
    internet
        |
   [ Ingress ]
   --|-----|--
   [ Services ]
```
の状態に変更したところまでできています。

ここでは  
1, Secretを作成する
2, ingress を TLS仕様にする  
という手順でhttpsにしていきます。

##### 2.1, Secretを作成する  
Secret は[ドキュメント](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls)にしたがっていきます。

```
apiVersion: v1
data:
  tls.crt: LS0tLS1CRUdJ..（省略）..S0tLQo=
  tls.key: LS0tLS1CRUdJ..（省略）..S0tLS0K
kind: Secret
metadata:
  name: test-app.com-ssl
  namespace: sandbox
type: Opaque
```
もちろん
```
$ kubectl apply -f /path/to/secret.yaml -n sandbox
```

##### 2.2, ingress を TLS仕様にする

作成した ingress を編集します。

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  namespace: sandbox
spec:
  tls:
  - secretName: test-app.com-ssl
  backend:
    serviceName: test-app
    servicePort: 8080
```
変更のコマンドもこちらで一発。
```
$ kubectl apply -f /path/to/ingress.yaml -n sandbox
```

すると `https://test-app.com` でアクセスできるようになります。  
ただ、`https://test-app.com`と`http://test-app.com`もアクセスできるので、リダイレクトの設定をしなければいけないです。
今回はそこまでやっていないので、割愛します。
