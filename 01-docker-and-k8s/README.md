## 1. Docker Compose から Kubernetes Manifest を作ってみよう

### 事前準備

このリポジトリをクローンして、`01-docker-and-k8s` のディレクトリに移動しておきます。

```bash
$ git clone https://github.com/inductor/docker-to-k8s.git
$ cd docker-to-k8s/01-docker-and-k8s
```

### Kompose の導入

Docker Compose の YAML から Kubernetes の YAML を生成するには [Kompose](https://github.com/kubernetes/kompose) を使います。まずは公式ドキュメントに沿ってインストールしてみましょう。

```bash
# Linux
$ curl -L https://github.com/kubernetes/kompose/releases/download/v1.26.0/kompose-linux-amd64 -o kompose

# macOS
$ curl -L https://github.com/kubernetes/kompose/releases/download/v1.26.0/kompose-darwin-amd64 -o kompose

$ chmod +x kompose
$ sudo mv ./kompose /usr/local/bin/kompose
```

導入できたら早速実行してみることにします。

```bash
$ kompose convert -f docker-compose.yaml -o k8s/ --with-kompose-annotation=false
WARN Volume mount on the host "/home/kela/ghq/src/github.com/inductor/docker-to-k8s/01-docker-and-k8s/data" isn't supported - ignoring path on the host
WARN Volume mount on the host "/home/kela/ghq/src/github.com/inductor/docker-to-k8s/01-docker-and-k8s/conf.d" isn't supported - ignoring path on the host
INFO Kubernetes file "k8s/minecraft-service.yaml" created
INFO Kubernetes file "k8s/nginx-service.yaml" created
INFO Kubernetes file "k8s/minecraft-deployment.yaml" created
INFO Kubernetes file "k8s/minecraft-claim0-persistentvolumeclaim.yaml" created
INFO Kubernetes file "k8s/nginx-deployment.yaml" created
INFO Kubernetes file "k8s/nginx-claim0-persistentvolumeclaim.yaml" created
```

なんか YAML が生成されましたね。では次に Kubernetes 環境を用意します

## Kind を使って Docker 上に Kubernetes を建てる

[kind](https://kind.sigs.k8s.io/) を使うと、Kubernetes を Docker 上に建てることができます。名前の通り "kind (Kubernetes-in-Docker)" になっていますね。

```bash
$ curl -L https://github.com/kubernetes-sigs/kind/releases/download/v0.11.1/kind-linux-amd64 -o kind
$ chmod +x kind
$ sudo mv ./kind /usr/local/bin/kind
```

導入できたらクラスターを作ります。

```bash
$ kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.21.1) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a nice day! 👋
```

環境ができたので k8s ディレクトリ配下に作られた YAML をとりあえず全部突っ込んでみます。

```
$ kubectl apply -f k8s/
persistentvolumeclaim/minecraft-claim0 created
deployment.apps/minecraft created
service/minecraft created
persistentvolumeclaim/nginx-claim0 created
deployment.apps/nginx created
service/nginx created
```

できちゃった！！便利。

```bash
$ kubectl get pod
NAME                         READY   STATUS    RESTARTS   AGE
minecraft-6cf746949f-d7g76   1/1     Running   0          116s
nginx-67885c6774-hvfjs       1/1     Running   0          116s
$ kubectl logs minecraft-6cf746949f-d7g76
[init] Changing ownership of /data to 1000 ...
[init] Running as uid=1000 gid=1000 with /data as 'drwxrwxrwx 2 1000 1000 4096 Dec 22 18:06 /data'
[init] Resolved version given 1.18.1 into 1.18.1
[init] Resolving type given SPIGOT
[init] Downloading Spigot from https://download.getbukkit.org/spigot/spigot-1.18.1.jar ...
[init] Downloading mod/plugin https://github.com/webbukkit/dynmap/releases/download/v3.3-beta-2/Dynmap-3.3-beta-2-spigot.jar ...
...
```

コンテナの起動ログも無事見えてますね。以下のように Service を見に行くと ClusterIP で公開されています。

```bash
$ kubectl get service
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)              AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP              3m23s
minecraft    ClusterIP   10.96.23.45     <none>        25565/TCP,8123/TCP   2m46s
nginx        ClusterIP   10.96.242.201   <none>        80/TCP,443/TCP       2m46s
```

次にコンテナの中に入ってマウントされているはずの conf を見てみましょう

```bash
$ kubectl exec -it nginx-67885c6774-hvfjs -- cat /etc/nginx/conf.d/default.conf
cat: /etc/nginx/conf.d/default.conf: No such file or directory
```

え、ないじゃん・・・。なんで？

無いのが普通です。Kubernetes ではコンテナに外部情報を注入するときは Volume を直接使うのではなく ConfigMap というものを使います。

```bash
$ kubectl create configmap nginx-config --from-file=conf.d/default.conf
```

ConfigMap は生成するとこんな感じになります。

```
$ kubectl describe configmap nginx-config
Name:         nginx-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
default.conf:
----
server {
    listen       80;
    server_name  localhost;

    location / {
    proxy_set_header      X-Real-IP $remote_addr;
    proxy_set_header      X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header      Host $http_host;
    proxy_redirect        off;
    proxy_read_timeout    6;

    # That is my dynmap webserver which is only accessible via localhost or through nginx
    proxy_pass            http://minecraft:8123;
    break;
    }
}


BinaryData
====

Events:  <none>
```

これを nginx から読み込むためには以下のような変更を YAML に加えます。

```diff
$ diff k8s/nginx-deployment.yaml k8s-patch/nginx-deployment-configmap.yaml
--- nginx-deployment.yaml       2021-12-23 13:20:22.035535873 +0900
+++ nginx-deployment-configmap.yaml     2021-12-23 13:21:46.935535979 +0900
@@ -29,10 +29,10 @@
           tty: true
           volumeMounts:
             - mountPath: /etc/nginx/conf.d
-              name: nginx-claim0
-      restartPolicy: Always
+              name: nginx-config-volume
       volumes:
-        - name: nginx-claim0
-          persistentVolumeClaim:
-            claimName: nginx-claim0
+        - name: nginx-config-volume
+          configMap:
+            name: nginx-config
+      restartPolicy: Always
 status: {}
```

Kubernetes 上に変更を適用する前に、実際の差分を計算するコマンドもあります。

```diff
$ kubectl diff -f k8s-patch/nginx-deployment-configmap.yaml
diff -u -N /tmp/LIVE-641766570/apps.v1.Deployment.default.nginx /tmp/MERGED-2103706877/apps.v1.Deployment.default.nginx
--- /tmp/LIVE-641766570/apps.v1.Deployment.default.nginx        2021-12-23 13:22:03.915536000 +0900
+++ /tmp/MERGED-2103706877/apps.v1.Deployment.default.nginx     2021-12-23 13:22:03.915536000 +0900
@@ -6,7 +6,7 @@
     kubectl.kubernetes.io/last-applied-configuration: |
       {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"creationTimestamp":null,"labels":{"io.kompose.service":"nginx"},"name":"nginx","namespace":"default"},"spec":{"replicas":1,"selector":{"matchLabels":{"io.kompose.service":"nginx"}},"strategy":{"type":"Recreate"},"template":{"metadata":{"creationTimestamp":null,"labels":{"io.kompose.service":"nginx"}},"spec":{"containers":[{"image":"nginx:mainline","name":"nginx","ports":[{"containerPort":80},{"containerPort":443}],"resources":{},"stdin":true,"tty":true,"volumeMounts":[{"mountPath":"/etc/nginx/conf.d","name":"nginx-claim0"}]}],"restartPolicy":"Always","volumes":[{"name":"nginx-claim0","persistentVolumeClaim":{"claimName":"nginx-claim0"}}]}}},"status":{}}
   creationTimestamp: "2021-12-22T18:06:34Z"
-  generation: 1
+  generation: 2
   labels:
     io.kompose.service: nginx
   managedFields:
@@ -15,6 +15,39 @@
     fieldsV1:
       f:metadata:
         f:annotations:
+          f:deployment.kubernetes.io/revision: {}
+      f:status:
+        f:availableReplicas: {}
+        f:conditions:
+          .: {}
+          k:{"type":"Available"}:
+            .: {}
+            f:lastTransitionTime: {}
+            f:lastUpdateTime: {}
+            f:message: {}
+            f:reason: {}
+            f:status: {}
+            f:type: {}
+          k:{"type":"Progressing"}:
+            .: {}
+            f:lastTransitionTime: {}
+            f:lastUpdateTime: {}
+            f:message: {}
+            f:reason: {}
+            f:status: {}
+            f:type: {}
+        f:observedGeneration: {}
+        f:readyReplicas: {}
+        f:replicas: {}
+        f:updatedReplicas: {}
+    manager: kube-controller-manager
+    operation: Update
+    time: "2021-12-22T18:06:48Z"
+  - apiVersion: apps/v1
+    fieldsType: FieldsV1
+    fieldsV1:
+      f:metadata:
+        f:annotations:
           .: {}
           f:kubectl.kubernetes.io/last-applied-configuration: {}
         f:labels:
@@ -67,48 +100,16 @@
             f:terminationGracePeriodSeconds: {}
             f:volumes:
               .: {}
-              k:{"name":"nginx-claim0"}:
+              k:{"name":"nginx-config-volume"}:
                 .: {}
-                f:name: {}
-                f:persistentVolumeClaim:
+                f:configMap:
                   .: {}
-                  f:claimName: {}
+                  f:defaultMode: {}
+                  f:name: {}
+                f:name: {}
     manager: kubectl-client-side-apply
     operation: Update
-    time: "2021-12-22T18:06:34Z"
-  - apiVersion: apps/v1
-    fieldsType: FieldsV1
-    fieldsV1:
-      f:metadata:
-        f:annotations:
-          f:deployment.kubernetes.io/revision: {}
-      f:status:
-        f:availableReplicas: {}
-        f:conditions:
-          .: {}
-          k:{"type":"Available"}:
-            .: {}
-            f:lastTransitionTime: {}
-            f:lastUpdateTime: {}
-            f:message: {}
-            f:reason: {}
-            f:status: {}
-            f:type: {}
-          k:{"type":"Progressing"}:
-            .: {}
-            f:lastTransitionTime: {}
-            f:lastUpdateTime: {}
-            f:message: {}
-            f:reason: {}
-            f:status: {}
-            f:type: {}
-        f:observedGeneration: {}
-        f:readyReplicas: {}
-        f:replicas: {}
-        f:updatedReplicas: {}
-    manager: kube-controller-manager
-    operation: Update
-    time: "2021-12-22T18:06:48Z"
+    time: "2021-12-23T04:22:03Z"
   name: nginx
   namespace: default
   resourceVersion: "681"
@@ -144,16 +145,17 @@
         tty: true
         volumeMounts:
         - mountPath: /etc/nginx/conf.d
-          name: nginx-claim0
+          name: nginx-config-volume
       dnsPolicy: ClusterFirst
       restartPolicy: Always
       schedulerName: default-scheduler
       securityContext: {}
       terminationGracePeriodSeconds: 30
       volumes:
-      - name: nginx-claim0
-        persistentVolumeClaim:
-          claimName: nginx-claim0
+      - configMap:
+          defaultMode: 420
+          name: nginx-config
+        name: nginx-config-volume
 status:
   availableReplicas: 1
   conditions:
```

よくわかりませんがそれっぽい差分がでましたね。Apply しちゃいましょう。

```bash
$ kubectl apply -f k8s-patch/nginx-deployment-configmap.yaml
deployment.apps/nginx configured
```

更新の様子を見てみましょう

```
$ kubectl get pod -n default
NAME                         READY   STATUS    RESTARTS   AGE
minecraft-6cf746949f-g77xm   1/1     Running   0          2m45s
nginx-594cb68ff7-svtf6       1/1     Running   0          4s
```

無事に Pod が入れ替わりました！！！！！やったね！！！

コンテナの中に config も無事に入ってそう。

```bash
$ kubectl exec -it nginx-594cb68ff7-svtf6 -- ls /etc/nginx/conf.d
default.conf
```

`kubectl port-forward` してブラウザ上で [http://localhost:8080](http://localhost:8080) にアクセスすると、無事 dynmap が見えました。

```
$ kubectl port-forward deployments/nginx 8080:80
```

終わったら Ctrl + C かなんかで終われば良い。

### お掃除

```bash
$ kind delete cluster
```

でさようなら
