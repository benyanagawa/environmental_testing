# environmental_testing

php_localはシンプルなコンテナ構成のPHP環境です。  
demoとしてplaneなlaravelをインストールしています。  

## ファイル構成
```
├── README.md
└── php_local
    ├── config
    │   ├── mysql
    │   │   └── local.cnf
    │   ├── nginx
    │   │   ├── default.conf
    │   │   └── nginx.conf
    │   └── php
    │       └── php.ini
    ├── db.yaml
    └── php.yaml
```

php.yaml db.yamlに記載されている内容がkubernetesでpodを作成するコンテナの詳細になります。  
configディレクトリはそれぞれのコンテナで必要なconfigを記述しています。  
後述にて行うconfigmapを使用するために必要となります。

## pod構成
構成は以下の通りです。

|pod| containar|
|:--|:--|
|local_php|php-fpm|
||nginx|
||memcached|
|mysql-deployment|mysql|

phpのpodには1つのpodに3つのコンテナが同居しています。  
mysqlのpodには1podで1コンテナになっています。  

## config作成
kubernetesにはconfigmapという機能が存在し、config情報をまとめて管理することができます。  
※ 以下コマンドはphp_localディレクトリ以下で行います。
```
$ kubectl create configmap nginx-config --from-file=config/nginx
$ kubectl create configmap mysql-config --from-file=config/mysql
$ kubectl create configmap php-config --from-file=config/php
```

上記の通りconfigmapをcreateすることにより以下のような形で管理されます。  
```
$ kubectl get configmap
NAME                  DATA   AGE
mysql-config          1      22h
nginx-config          2      147m
php-config            1      22h
```

このconfigmapでつけた名前はphp.yamlやdb.yamlで使用します。  

## 環境立ち上げ
configmapを作ったら環境を立ち上げます。  
```
$ kubectl apply -f php.yaml -f db.yaml
```
環境が立ち上がると以下のようになります。
```
$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
local-php-8696545899-lkrcp       3/3     Running   0          78m
mysql-deployment-f5bc645-6kvps   1/1     Running   0          144m
```

環境が立ち上がったら  
http://localhost:82/  
上記アドレスへアクセスするとphp_local/app以下のlaravelのwelcome画面が表示されます。

正しく立ち上がらない場合は
```
$ kubectl describe [pod name]
```
にてpodの状況を確認してください。

## scele
kubernetesではservice(Load Balancer), deployment, podの構成となっており、deploymentにてpodの情報を更新させます。  
その中でもアクセス数によって手動で（自動での設定も可能です）自動の場合、heapster等のインストール、AWSであればAutoScalingGroup等が使用可能です。  
今回は手動でpodのscaleが可能なことを確認します。  

まずはdeployment名を取得します。
```
$ kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
local-php          2/2     2            2           89m
mysql-deployment   1/1     1            1           154m
```
phpのdeploymentはlocal-php(php.yamlに記載されています)となるので、以下のコマンドを実行することによりScaleが可能となります。
```
$ kubectl scale deployment local-php --replicas=5
deployment.extensions/local-php scaled
```
上記はreplicasを5に変更し、podの数を増やします。
```
$ kubectl get pods
NAME                             READY   STATUS              RESTARTS   AGE
local-php-8696545899-2qjxb       0/3     ContainerCreating   0          7s
local-php-8696545899-fkx25       0/3     ContainerCreating   0          7s
local-php-8696545899-lkrcp       3/3     Running             0          92m
local-php-8696545899-sxtbs       0/3     ContainerCreating   0          7s
local-php-8696545899-t6pmr       0/3     ContainerCreating   0          7s
mysql-deployment-f5bc645-6kvps   1/1     Running             0          157m
```
podが5つに別れ、service(Load Balancer)にて振り分けがされるようになりました。  
また、imageをECRやDockerhubにアップすることにより、podのimageを変更することも可能です。

## 環境削除
kubernetesでは前述の通り、deployment, service, podで構成されており、deploymentが管理を行っています。  
そのため、podのみを削除しようとしてもreplicas分のpodをdeploymentが作成しようとします。  
環境を削除する場合はphp_localディレクトリ以下にて
```
$ kubectl delete -f php.yaml -f db.yaml
```
上記コマンドを実行することにより、deployment, service, podを削除することができます。
