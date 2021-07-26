# Overview
フロントWEBアプリケーションをkubernetes上で動作させる為のmanifest一式になります。

# Details
### kustomization.yaml
kubernetes上に反映するリソースを定義しています。<br>
基本的にはbuildするイメージをセットし、buildして利用します。<br>
イメージセットの例
```
kustomize edit set image back-api=quay.io/tyonesaka/back-api:xxxx
```
ビルドの例
```
kustomize build -o resource.yam
```
applyの例
```
kubectl apply -f resource.yaml
```

### deployment.yaml
PODやReplica Setを管理するオブジェクトを定義しています。
#### 重要事項抜粋
deployment名
```
  3 metadata:
  4   name: back-api
```
Replica set数
```
 12   replicas: 1
```
POD名
```
 13   template:
 14     metadata:
 15       labels:
 16         app: back-api
```
アプリケーションコンテナ名
```
 19       containers:
 20       - name: back-api
```
アプリケーションコンテナのリソース
```
 47         resources:
 48           requests:
 49             memory: "1Gi"
 50             cpu: "1"
 51           limits:
 52             memory: "1Gi"
 53             cpu: "1"
```
アプリケーションコンテナのポート
```
 54         ports:
 55         - containerPort: 8000
```
secretを環境変数に設定<br>
APIを実行する為の認証情報をsecretに登録し、環境変数として扱う。
```
 26         env:
 27         - name: API_KEY
 28           valueFrom:
 29             secretKeyRef:
 30               name: wd-secret
 31               key: apikey
 32         - name: COLLECTION_ID
 33           valueFrom:
 34             secretKeyRef:
 35               name: wd-secret
 36               key: collectionID
 37         - name: ENVIRONMENT_ID
 38           valueFrom:
 39             secretKeyRef:
 40               name: wd-secret
 41               key: environmentID
 42         - name: SERVICE_URL
 43           valueFrom:
 44             secretKeyRef:
 45               name: wd-secret
 46               key: serviceUrl
```
preStopフックでのSleep<br>
preStopフックで一定時間の sleep を行うと、コンテナが停止される前に指定した時間だけ待機する動作になります。<br>
これによって、Podへのトラフィックの配送が止まってから（サービスアウトしてから）終了処理に入るようにすることができます。
```
 21         lifecycle:
 22           preStop:
 23             exec:
 24               command: ["sh", "-c", "sleep 30"]
```
imageの設定はkustomizuコマンドで行う為、Keyを指定
```
 25         image: back-api # Overwrite it with the kustomize command.
```

## service.yaml
Podへの接続を解決してくれる抽象的なオブジェクト<br>
Selectorで選択したPodをUpstreamとしたLBみたいなもの<br>
#### 重要事項抜粋
Label SelectorでトラフィックをルーティングするPodを選択する。
```
 13   selector:
 14     app: back-api
```
portsに転送設定を設定。
```
  8   ports:
  9   - name: http
 10     port: 80
 11     protocol: TCP
 12     targetPort: 8000
```

## route.yaml
serviceを外部からアクセス出来るようにするためのOpenShiftのオブジェクト<br>
対象のサービスを指定
```
  8   to:
  9     kind: Service
 10     name: back-api
```