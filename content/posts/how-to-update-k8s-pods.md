---
title: "Kubernetes の pods を更新する方法"
date: 2018-03-28T09:08:18+09:00
draft: false
slug: how-to-update-k8s-pods
---


ローカルからDocker image を Build して、 GCRにpush, k8sのアプリケーションを変更する方法を記述します。


## Build
    
    docker build --rm -t asia.gcr.io/sample-project/example:v1 .

`--rm` はビルド成功後中間コンテナを削除するオプションです。`-t`はタグをイメージにつけることができます。  
この場合は v1を `asia.gcr.io/sample-project/example` に付与しています。

## Test (RUN)
    
    docker run --rm -it -P asia.gcr.io/sample-project/example:v1 .
    
インタラクティブなプロセスでは、コンテナのプロセスに対して tty を割り当てるために、 -i -t を一緒に使う必要があります。

## Upload
    
    gcloud docker -- push asia.gcr.io/sample-project/example:v1
    
gcloud をインストールしている必要があります。

## Deploy
    
    kubectl set image deployment/example example=asia.gcr.io/sample-project/example:v1 -n your-namespace
    kubectl rollout status deployment/example -n your-namespace
    
`set image` で deployment に新しいイメージを付与して、 `rollout` コマンドでpodの更新をします。