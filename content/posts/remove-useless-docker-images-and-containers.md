---
title: "Dockerで開発している時に不要になった image や container を全て削除する"
date: 2018-03-28T09:21:40+09:00
draft: false
slug: remove-useless-docker-images-and-containers
---


Docker を使っていると、docker 関係のファイルがたくさん生成され、PCのvolumeを圧迫していることがあります。  
そんな時は以下のコマンドを適宜活用すると数GB減らすことができるのでとても便利です。

## container の終了
    
    $ docker kill [containerID]

## 不要な docker container の削除
    
    $ docker rm $(docker ps -f status=exited -q)

## 不要な docker image の削除
    
    $ docker rmi $(docker images -f dangling=true -q)

## container に入る
    
    $ docker exec -it [containerID] /bin/bash

## ログを見る
    
    $ docker logs -f [containerID]