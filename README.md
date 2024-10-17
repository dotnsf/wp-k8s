# WordPress on Kubernetes(OCP)

Kubernetes の機能を最大限（？）活かして WordPress 環境を作る


## WordPress の特徴

- PHP の CMS ウェブアプリケーション

- 投稿データは MySQL データベースに格納
  - Volume 内に格納するべき
  - 

- 画像などのアップロードファイルはファイルシステム内に格納
  - 上とは別の Volume 内に格納するべき
  - ウェブアプリケーションをスケールアウトした時に同一ファイルを参照できるようにする


## k8s 向けのベストプラクティスっぽいもの

- ウェブアプリケーションはスケーラブルにするべき
  - そのためのコンテナクラスタ環境

- Volume を活用するべき
  - Volume は Deployment や Pod とは独立していて、Deployment/Pod が消えても残る
  - データ類は Volume に集めるべき
  - 具体的には以下２つの Volume を定義して使う
    - DB 用 Volume
      - MySQL のデータ（/var/lib/mysql）を格納するための Volume
    - App 用 Volume
      - WordPress のデータ（アップロードファイルを含む /var/www/html）を格納するための Volume
  - コンテナクラスタ環境なので、ディレクトリボリュームは使えない
    - `-v /tmp/mysql:/var/lib/mysql` のような指定が使えるのはシングルノード内だけ
    - `-v mysqlvolume:/var/lib/mysql` のようにあらかじめ用意したボリューム名で指定するべき

- MySQL のパスワード管理は secret を使うべき
  - k8s の secret で MySQL パスワードを一元管理し、DB Pod と App Pod から参照して使えるようにする

 
## 上記 "ベストプラクティスっぽいもの" を実践する

- MySQL のパスワード管理用 secret : `wp-secret.yaml`
  - `$ oc create secret generic wordpress --from-literal=db-password=P@ssw0rd --dry-run=client -o yaml > wp-secret.yaml`
    - db-password の値は適当に変える
  - `$ vi wp-secret.yaml` で以下のように編集する

```(wp-secret.yaml)
apiVersion: v1
data:
  db-password: UEBzc3cwcmQ=
kind: Secret
metadata:
  name: wordpress
```

- MySQL 用 PVC : `mysql-pvc.yaml`
  - `$ vi mysql-pvc.yaml` で以下の内容のファイルを作成する
  - 最下行のストレージサイズは適当に指定する

```(mysql-pvc.yaml)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

- アップロードファイル用 PVC : `wp-pvc.yaml`
  - `$ vi wp-pvc.yaml` で以下の内容のファイルを作成する
  - 最下行のストレージサイズは適当に指定する

```(wp-pvc.yaml)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

- MySQL 用 Deployment : `mysql-deployment.yaml`
  - `$ oc create deployment mysql --image=mysql --dry-run=client -o yaml > mysql-deployment.yaml`
  - `$ vi mysql-deployment.yaml` で以下のように編集する
    - `wp-secret.yaml` のシークレットから DB パスワードを取得
    - `mysql-pvc.yaml` の PVC を指定して永続化

```(mysql-deployment.yaml)
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: mysql
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - image: mysql
          name: mysql
          env:
            - name: MYSQL_RANDOM_ROOT_PASSWORD
              value: "yes"
            - name: MYSQL_DATABASE
              value: wpdb
            - name: MYSQL_USER
              value: wpuser
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: wordpress
                  key: db-password
          volumeMounts:
            - mountPath: /var/lib/mysql
              name: mysql-volume
      volumes:
        - name: mysql-volume
          persistentVolumeClaim:
            claimName: mysql-pvc
```

- MySQL 用 Service : `mysql-service.yaml`
  - ここまでに作ったマニフェストからリソースを生成する
    - `$ oc apply -f .`
  - MySQL の Deployment の情報を使って、公開するための Service マニフェストの雛形を作る
    - `$ oc expose deployment mysql --name=mysql-service --port=3306 --dry-run=client -o yaml > mysql-service.yaml`
  - `$ vi mysql-service.yaml` で以下のように編集する
  - 編集後にリソースを生成する
    - `$ oc apply -f mysql-service.yaml`

```(mysql-service.yaml)
apiVersion: v1
kind: Service
metadata:
  labels:
    app: mysql
  name: mysql-service
spec:
  ports:
  - port: 3306
    protocol: TCP
    targetPort: 3306
  selector:
    app: mysql
```

- WordPress 用 Deployment : `wp-deployment.yaml`
  - `$ oc create deployment wordpress --image=wordpress --dry-run=client -o yaml > wp-deployment.yaml`
  - `$ vi wp-deployment.yaml` で以下のように編集する
    - `wp-secret.yaml` のシークレットから DB パスワードを取得
    - `wp-pvc.yaml` の PVC を指定して永続化
  - 編集後にリソースを生成する
    - `$ oc apply -f wp-deployment.yaml`

```(wp-deployment.yaml)
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: wordpress
  name: wordpress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
        - image: wordpress
          name: wordpress
          env:
            - name: WORDPRESS_DB_HOST
              value: mysql-service
            - name: WORDPRESS_DB_USER
              value: wpuser
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: wordpress
                  key: db-password
            - name: WORDPRESS_DB_NAME
              value: wpdb
          volumeMounts:
            - mountPath: /var/www/html
              name: wp-volume
      volumes:
        - name: wp-volume
          persistentVolumeClaim:
            claimName: wp-pvc
```

- WordPress 用 Service : `wp-service.yaml`
  - WordPress の Deployment の情報を使って、公開するための Service マニフェストの雛形を作る
    - `$ oc expose deployment wordpress --port=80 --dry-run=client -o yaml > wp-service.yaml`
  - `$ vi wp-service.yaml` で以下のように編集する
  - 編集後にリソースを生成する
    - `$ oc apply -f wp-service.yaml`

```(wp-service.yaml)
apiVersion: v1
kind: Service
metadata:
  labels:
    app: wordpress
  name: wordpress-service
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: wordpress
```


## 動作確認

- ポートフォワードで動作確認
  - `$ oc port-forward service/wordpress-service 8080:80`
  - `http://localhost:8080/`


## クリーンアップ

- 全リソース削除
  - `$ oc delete -f .`


## Reference

- 順を追ってkubernetes上でWordPressを展開する
  - https://qiita.com/moreyhat/items/02d8a6d6c9791cff9a1d
  - 上の `App 用 Volume` 以外のものを紹介している

