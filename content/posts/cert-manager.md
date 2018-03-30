---
title: "Kubernetes 上で Cert Manager の設定の仕方"
date: 2018-03-28T16:39:37+09:00
draft: false
slug: "cert-manager-on-k8s"
---

このブログでは Cluster の作成からアプリケーションのデプロイ、Ingress と Ingress Nginx Controller の設定、
Cert-manager の導入と Let's encrypt 導入を行います。  
どちらかというと、Cert-manager の導入と Let's encrypt 導入がメインですが、Kubernetes を使って、アプリケーションを動かす時に、
SSL/TLSの設定や Nginx の細かい設定を行えるようなチュートリアルになっています。

## Cluster の作成

[Google Cloud Console](https://console.cloud.google.com/)から cluster を作成します。  
メニュー > Kubernetes Engine から クラスタを作成 を押します。

この際に `以前の承認` というところを有効にしておかないと、Kubernetes Dashboard をみるのに別の設定が必要になります。  

## アプリケーションのデプロイ

好きなアプリケーションを作ってサクッとデプロイしてください。  
例えばこんなアプリを作成します。
```yaml

apiVersion: v1
kind: Service
metadata:
  name: exmaple-app
  namespace: ltd
  labels:
    app: exmaple-app
spec:
  type: LoadBalancer
  ports:
  - port: 8080
    targetPort: 8080
    protocol: TCP
  selector:
    app: exmaple-app
---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: exmaple-app
  namespace: default
spec:
  replicas: 1
  # Deployment ammount of history to keep
  revisionHistoryLimit: 3
  template:
    metadata:
      namespace: default
      labels:
        app: exmaple-app
        version: latest
    spec:
      containers:
      - name: exmaple-app
        image: asia.gcr.io/sample-project/exmaple-app:v1
        ports:
        - containerPort: 8080

```

k8s 上のアプリケーションの作成方法は [公式サイト](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)が詳しいです。


## Ingress の設定

[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)はとても強力なツールです。
HTTPリクエストを上手く捌いてくれます。 Ingress を設定するに当たって、まずは LoadBalancer から NordPort を使うように変更します。 

```yaml

apiVersion: v1
kind: Service
metadata:
  name: exmaple-app
  namespace: ltd
  labels:
    app: exmaple-app
spec:
  type: NordPort   <= ここを変更
  ports:
  - port: 8080
    targetPort: 8080
    protocol: TCP
  selector:
    app: exmaple-app
---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: exmaple-app
  namespace: default
spec:
  replicas: 1
  # Deployment ammount of history to keep
  revisionHistoryLimit: 3
  template:
    metadata:
      namespace: default
      labels:
        app: exmaple-app
        version: latest
    spec:
      containers:
      - name: exmaple-app
        image: asia.gcr.io/sample-project/exmaple-app:v1
        ports:
        - containerPort: 8080

```


ingress 用のファイルを書きます。今回は `ingress.yaml` で保存します。

```yaml

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-example-com
  namespace: default
  annotations:
    kubernetes.io/ingress.class: gce
spec:
  rules:
  - host: example.com
    http:
      paths:
      - backend:
          serviceName: exmaple-app
          servicePort: 8080

```

保存したら cluster にあげます。

    kubectl create -f ingress.yaml 
    
   

## Ingress Controller の設定

GCP上で ingress をあげると、デフォルトでは GCE という Ingress Controller が割り当てられます。 
ただ、nginx 上の細かい設定は [Ingress nginx controller](https://github.com/kubernetes/ingress-nginx) のほうが得意なので、設定していきます。 

Ingress nginx controller を設置すると、Ingress に設定した条件を介して、nginx を使うようになります。

![](https://storage.googleapis.com/gcp-community/tutorials/nginx-ingress-gke/Nginx%20Ingress%20on%20GCP%20-%20Fig%2002.png)
https://cloud.google.com/community/tutorials/nginx-ingress-gke より


Ingress nginx controllerをデプロイする方法は[こちら](https://github.com/kubernetes/ingress-nginx/blob/master/deploy/README.md)が詳しいです。

### Kubernetes Engine に Ingress nginx controller を deploy する。

以下のコマンドを入力します。
```bash
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/namespace.yaml \
    | kubectl apply -f -

curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/default-backend.yaml \
    | kubectl apply -f -

curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/configmap.yaml \
    | kubectl apply -f -

curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/tcp-services-configmap.yaml \
    | kubectl apply -f -

curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/udp-services-configmap.yaml \
    | kubectl apply -f -
```

次にRBAC roles という権限付きでデプロイする場合は以下のコマンドを

```bash
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/rbac.yaml \
    | kubectl apply -f -

curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/with-rbac.yaml \
    | kubectl apply -f -
```

そうでない場合は以下のコマンドを入力します。
```bash
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/without-rbac.yaml \
    | kubectl apply -f -
```

Kubernetes Engine (GCE - GKE) では以下のコマンドを入力します。
```bash
kubectl patch deployment -n ingress-nginx nginx-ingress-controller --type='json' \
  --patch="$(curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/publish-service-patch.yaml)"
```

次にRBAC なら
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/patch-service-with-rbac.yaml
```

そうでないなら
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/patch-service-without-rbac.yaml
```

きちんとデプロイできたら、ingress も変更しておきます。

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-example-com
  namespace: default
  annotations:
    kubernetes.io/ingress.class: nginx  <= ここを gce から nginx に変更
spec:
  rules:
  - host: example.com
    http:
      paths:
      - backend:
          serviceName: exmaple-app
          servicePort: 8080
```

そうすると、ingress-nginx という名前で service が作られます。
外部IPが振り当てられるので、DNSでそのIPを設定します。

```
              internet
                 |
   [ Ingress Nginx Controller ] ⇄ [ Ingress ]
            --|-----|--
            [ Services ]
```

[ Ingress Nginx Controller ] と [ Ingress ] と両方に IP が当てられていますが、
[ Ingress ]のIPを使うと GCE の Ingress Controller を
[ Ingress Nginx Controller ]のIPを使うと Ingress Nginx Controller を
使うようになります。



## Cert-manager の設定
[cert-manager](https://github.com/jetstack/cert-manager/)は TLS証明書を Kubernetes で自動取得・更新するためのサードパーティです。
![](https://raw.githubusercontent.com/ahmetb/gke-letsencrypt/master/img/gke-letsencrypt.png)  
この[チュートリアル](https://github.com/ahmetb/gke-letsencrypt)がかなり役に立ちます。


### helm のインストール

[公式サイト](https://docs.helm.sh/using_helm/#installing-helm)に従って 
Kubernetes のパッケージマネージャである[helm](https://docs.helm.sh/) をローカルにインストールします。  
MacOSを使っている場合は以下のコマンドが一番楽です。

    brew install kubernetes-helm
    
その他バイナリからも、スクリプトからもインストールが可能です。

次に、Tiller と呼ばれる Helm のサーバーサイドコンポーネントを現在アプリが動いている Kubernetes cluster にインストールします。

    kubectl create serviceaccount -n kube-system tiller
    
    kubectl create clusterrolebinding tiller-binding \
        --clusterrole=cluster-admin \
        --serviceaccount kube-system:tiller
    
    helm init --service-account tiller

以上のコマンドを走らせて、Deployment名が `tiller-deploy` の tiller pod が使用可能になったら以下のコマンドを走らせます。

    helm repo update


### cert-manager のインストール

次に[cert-manager](https://github.com/jetstack/cert-manager/)をインストールします。  
helm コマンドをつかって、cert-manager のレポジトリをインストールし、 helm chart を Kubernetes cluster にデプロイします。

    helm install --name cert-manager \
        --namespace kube-system stable/cert-manager

### let's encrypt の設定

cert-manager で管理できる TLS 証明書はいくつもあるのですが、今回は Let's Encrypt を使用していきます。  
まずは証明書発行者のメールアドレスを 環境変数として設定します。

    export EMAIL=toon@example.com

次に以下のコマンドを走らせます。

    curl -sSL https://rawgit.com/ahmetb/gke-letsencrypt/master/yaml/letsencrypt-issuer.yaml | \
        sed -e "s/email: ''/email: $EMAIL/g" | \
        kubectl apply -f-


これの中身は staging 用と production 用の ClusterIssuer を作成するコマンドになっていて、
空欄になっているメールアドレスに先ほど登録したメールアドレスが設定されます。

```yaml
# https://rawgit.com/ahmetb/gke-letsencrypt/master/yaml/letsencrypt-issuer.yaml

apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging.api.letsencrypt.org/directory
    email: ''
    privateKeySecretRef:
      name: letsencrypt-staging
    http01: {}
---
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v01.api.letsencrypt.org/directory
    email: ''
    privateKeySecretRef:
      name: letsencrypt-prod
    http01: {}

```
うまく作成されると以下のアウトプットが表示されます。

    clusterissuer "letsencrypt-staging" created
    clusterissuer "letsencrypt-prod" created


もし、`no matches for certmanager.k8s.io/, Kind=ClusterIssuer` というエラーが返ってきた場合は、
[cert-manager のインストール](#cert-manager-のインストール) からやり直してください

cert-manager の素晴らしいところは kube-lego で不可能だった Let's Encrypt のステージング、プロダクションのエンドポイントの区別を管理できます。  

さて今回は `letsencrypt-prod` を使って証明書を取得していきます。

### 証明書の取得


certificate.yaml として以下のファイルを保存します。

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: example-com-tls
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: example.com
  dnsNames:
  - example.com
  acme:
    config:
    - http01:
        ingress: ingress-example-com
      domains:
      - example.com
```
ただし、
matadata.name と spec.secretName の `example-com-tls` は好きな名前に、  
spec.commonName, dnsNames, acme.config.domains の `example.com` は取得したいドメインに、   
ingress名の `ingress-example-com` はアプリケーションが動いている ingress の名前に変更します。

保存したら、 cluster に反映させます。

    kubectl apply -f certificate.yaml
    
しばらくしてから証明書が取れているかを確認します。

    $ kubectl get secrets
    NAME              TYPE                DATA      AGE
    example-com-tls   kubernetes.io/tls   2         1m

証明書の取得情報をみてみましょう。

    $ kubectl describe certificate example-com-tls 
    
    ...
    Type     Reason                Message
    ----     ------                -------
    Warning  ErrorCheckCertificate Error checking existing TLS certificate: secret "example-com-tls" not found
    Normal   PrepareCertificate    Preparing certificate with issuer
    Normal   PresentChallenge      Presenting http-01 challenge for domain foo.kubernetes.tips
    Normal   SelfCheck             Performing self-check for domain example.com
    Normal   ObtainAuthorization   Obtained authorization for domain example.com
    Normal   IssueCertificate      Issuing certificate...
    Normal   CeritifcateIssued     Certificated issued successfully
    Normal   RenewalScheduled      Certificate scheduled for renewal in 1438 hours
    
`Certificate scheduled for renewal in 1438 hours` と書かれているように、 1438時間後（約２ヶ月後）に再取得すると書かれています。  

さて、ここまでできちんと証明書の取得＆自動更新設定ができたので、実際に https で動かしてみましょう！

### HTTPS でアプリケーションを動かす

すでにある ingress のファイルを以下のように変更します。

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-example-com
  namespace: ltd
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
    - hosts:
      - example.com
      secretName: example-com-tls  <=ここを取得した証明書の名前にする
  rules:
  - host: example.com
    http:
      paths:
      - backend:
          serviceName: example
          servicePort: 8080
```

保存したら cluster に反映させます。

    kubectl apply -f ingress-example-com.yaml

https://exmaple.com にアクセスして確認してみましょう！！  
きちんと設定できていたら完了です。


## 参考

- [cert-manager](https://github.com/jetstack/cert-manager/)  
- [gke-letsencrypt](https://github.com/ahmetb/gke-letsencrypt)  
- [GKE で TLS 証明書を自動管理(cert-manager DNS-01 編)](https://qiita.com/apstndb/items/3a39a1e6acacbbc30765)