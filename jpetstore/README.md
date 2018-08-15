JpetstoreをKubernetesにデプロイしてみよう

<!-- TOC -->

- [コードの取得](#コードの取得)
- [環境準備](#環境準備)
- [アプリのビルドとレジストリへの登録](#アプリのビルドとレジストリへの登録)
- [Kubernetesへデプロイ](#kubernetesへデプロイ)
- [アプリへアクセス](#アプリへアクセス)
- [tips](#tips)

<!-- /TOC -->


## コードの取得
1. サンプルコードをクローン
  
    ```
    git clone git@github.com:kota661/jpetstore-kubernetes.git
    cd jpetstore-kubernetes
    ```

## 環境準備
1. ibmcloud cliでログイン

    ```
    # IBM Cloudへのログイン
    ic login -a https://api.ng.bluemix.net

    # container serviceが利用する地域を設定
    ic cs region-set us-south

    # kubernetesの接続情報の取得
    # コマンド実行後に環境変数 KUBECONFIG の設定情報が出力されるためその行をコピペしてください
    # ex: export KUBECONFIG=/Users/$USER/.bluemix/plugins/container-service/clusters/mycluster/kube-config-dal12-mycluster.yml
    ibmcloud cs cluster-config mycluster

    # kubernetesの接続情報確認
    kubectl get nodes
    ```

2. Registryの作成

    ```
    # login
    ibmcloud cr login

    # 作成済みのスペースがあるか確認
    # 作成済みのスペースが有る場合は、imageが保管されていないか確認。
    # イメージの確認：ibmcloud cr image-list
    # イメージの削除：ibmcloud cr image-rm registry.ng.bluemix.net/kota/hello-k8s:v1
    ibmcloud cr namespaces

    # スペースの作成
    # ex: ibmcloud cr namespace-add kota
    ibmcloud cr namespace-add <NAMESPACE>

    # このあとのコマンドが楽になるように環境変数の設定
    # ex: export MYNAMESPACE=kota
    # ex: export MYREGISTRY=registry.ng.bluemix.net
    # ※地域によって異なるため、ibmcloud cr info コマンドにて確認ください
    export MYNAMESPACE=<NAMESPACE>
    export MYREGISTRY=<REGISTRY>
    ```

## アプリのビルドとレジストリへの登録
1. jpetstoreのビルドとレジストリへの登録

    ```
    # jpetstore-kubernetes/jpetstore
    cd jpetstore

    # docker build
    docker build . -t ${MYREGISTRY}/${MYNAMESPACE}/jpetstoreweb

    # buildしたイメージの確認
    docker run -d -p 9080:9080 ${MYREGISTRY}/${MYNAMESPACE}/jpetstoreweb:latest

    # buildされたimageの確認
    docker images

    # imageをIBM CloudのRegistryに登録
    docker push ${MYREGISTRY}/${MYNAMESPACE}/jpetstoreweb

    # 登録結果の確認
    ibmcloud cr image-list
    ```

1. dbのビルド

    ```
    # jpetstore-kubernetes/jpetstore/db
    cd db

    # docker build
    docker build . -t ${MYREGISTRY}/${MYNAMESPACE}/jpetstoredb

    # buildされたimageの確認
    docker images

    # imageをIBM CloudのRegistryに登録
    docker push ${MYREGISTRY}/${MYNAMESPACE}/jpetstoredb

    # 登録結果の確認
    ibmcloud cr image-list
    ```


## Kubernetesへデプロイ

1. cd 

    ```
    # jpetstore-kubernetes/jpetstore
    cd ..
    ```

2. yamlの変更

    Ingressのドメイン名と、DockerImageのレジストリを変更
  
    * 変更するファイル
      - jpetstoredb-deployment.yaml
      - jpetstoreweb-deployment.yaml
      - jpetstoreweb-ingress.yaml
    * 変更箇所
      * `<CLUSTER DOMAIN>`
        * `ibmcloud cs cluster-get mycluster` を実行しIngress Subdomainと記載されている箇所を入力
        * ex: mycluster-726465.us-south.containers.appdomain.cloud
      * `<REGISTRY NAMESPACE>`
        * ex: image: registry.ng.bluemix.net/kota/jpetstoreweb

3. yamlを適用しデプロイ

    ```
    kubectl apply -f jpetstoredb-deployment.yaml
    kubectl apply -f jpetstoredb-service.yaml
    kubectl apply -f jpetstoreweb-deployment.yaml
    kubectl apply -f jpetstoreweb-service.yaml
    ```

4. サービス公開
  * Ingress
  
    ```
    kubectl apply -f jpetstoreweb-ingress.yaml
    ```
  
  * NodePort

    ```
    kubectl apply -f jpetstoreweb-nodeport.yaml
    # worker nodeのIPを確認
    ibmcloud cs workers mycluster
    ```

## アプリへアクセス

* Ingress

  http://jpetstore.<CLUSTER DOMAIN>  
  ex: http://jpetstore.mycluster-726465.us-south.containers.appdomain.cloud


* NodePort
  
  http://<WORKER NODE PUBLIC IP>:<SVC PORT>  
  ex: http://69.56.28.251:30060


## tips
* cs, crプラグインがインストールされていない
  
  ```
  ibmcloud plugin install container-service -r Bluemix
  ibmcloud plugin install container-registry -r Bluemix
  ```

* docker push で認証エラー
  
  `ibmcloud cr login`

* crが容量いっぱいのため新規で追加できない
  * 以下Docker imageを利用ください
  * cloudhandson/jpetstoreweb:latest
  * cloudhandson/jpetstoredb:latest