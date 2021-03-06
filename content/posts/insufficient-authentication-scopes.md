---
title: "GCPで 「Request had insufficient authentication scopes.」と怒られた時の対処法"
date: 2018-02-07T18:49:38+09:00
draft: false
---

[GCEでオープンソースの Drone.io](https://github.com/drone/drone) を動かしていて、そこから GKE にアクセスしています。  
今回新しく cluster を作成して、そこにdroneのGCEからアクセスしようと思ったのですが、以下のように怒られました。

```
$ gcloud container clusters get-credentials test-cluster
(gcloud.container.clusters.get-credentials) ResponseError: code=403, message=Request had insufficient authentication scopes."
```

色々探してみたのですが、このエラーは簡単にいうと **パーミッションがない** ということです。  
authentication scopes というのはサービスアカウントやユーザーによってGCPのAPIへのアクセス権がないということに等しい（と思います）  

今回は一旦VMを止めて、編集から上記の Cloud API access scopes を適切なものに変更して直りました。
