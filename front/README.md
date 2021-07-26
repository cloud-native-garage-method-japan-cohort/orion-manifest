# Overview
フロントWEBアプリケーションをkubernetes上で動作させる為のmanifest一式になります。

# Details
### kustomization.yaml
kubernetes上に反映するリソースを定義しています。<br>
基本的にはbuildするイメージをセットし、buildして利用します。<br>
イメージセットの例
```
kustomize edit set image front-app=quay.io/tyonesaka/front-app:xxxx
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
  4   name: front-app
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
 16         app: front-app
```
アプリケーションコンテナ名
```
 19       containers:
 20       - name: front-app
```
アプリケーションコンテナのリソース
```
 26         resources:
 27           requests:
 28             memory: "2Gi"
 29             cpu: "1"
 30           limits:
 31             memory: "2Gi"
 32             cpu: "1"
```
アプリケーションコンテナのポート
```
 33         ports:
 34         - containerPort: 8081
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
 25         image: front-app # Overwrite it with the kustomize command.
```

## service.yaml
Podへの接続を解決してくれる抽象的なオブジェクト<br>
Selectorで選択したPodをUpstreamとしたLBみたいなもの<br>
#### 重要事項抜粋
Label SelectorでトラフィックをルーティングするPodを選択する。
```
 13   selector:
 14     app: front-app
```
portsに転送設定を設定。
```
  8   ports:
  9   - name: http
 10     port: 80
 11     protocol: TCP
 12     targetPort: 8081
```

## route.yaml
serviceを外部からアクセス出来るようにするためのOpenShiftのオブジェクト<br>
対象のサービスを指定
```
  8   to:
  9     kind: Service
 10     name: front-app
```