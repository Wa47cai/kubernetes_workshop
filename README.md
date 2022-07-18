# Kubenates workshop

* [Kubenates workshop](#kubenates-workshop)
    * [准备工作](#准备工作)
        * [安装 docker](#安装-docker)
        * [安装 minikube](#安装-minikube)
        * [安装 k8s CLI 和 Terminal based UI](#安装-k8s-cli-和-terminal-based-ui)
        * [注册 docker hub 账号登录](#注册-docker-hub-账号登录)
    * [Container](#container)
    * [Pod](#pod)
    * [Deployment](#deployment)
        * [release new verison](#release-new-verison)
        * [Rolling Update](#rolling-update)
        * [Automatically blocking bad releases by readinessProbe](#automatically-blocking-bad-releases-by-readinessprobe)
    * [Service](#service)
    * [ingress](#ingress)
    * [Configmap](#configmap)
        * [env var](#env-var)
        * [configmap](#configmap-1)
        * [Getting all the variables from a ConfigMap](#getting-all-the-variables-from-a-configmap)
        * [Exposing ConfigMap as files](#exposing-configmap-as-files)
    * [Secret](#secret)
        * [Using stringData](#using-stringdata)
    * [helm](#helm)

## 准备工作

本地环境是 MacOS 11.6.2 Intel 版本，教程也是本地环境。windows 或 linux 环境的小伙伴需要自行安装 docker 和 minikube。

如果本地

### 安装 docker

因为 docker desktop 各种协议和法律问题，已经不建议大家直接安装 desktop 使用，建议只安装 CLI。注意：如果本地已经安装了 docker desktop，那么可以忽略这一步，下一步也可以简化。

```bash
# Install Docker CLI
brew install docker
brew install docker-compose
```

### 安装 minikube

[minikube](https://minikube.sigs.k8s.io/docs/) 用于在本地环境中运行 Kubernetes 集群。但它也运行一个可用于运行容器的 docker 守护进程。在 macOS 上，minikube 运行在很多虚拟化技术上，可以选择[hyperkit](https://minikube.sigs.k8s.io/docs/drivers/hyperkit/)，这里因为我本地之前已经安装过 virtualbox (brew install --cask virtualbox)，所以我用的是 virtualbox 虚拟化技术。如果你本地之前已经安装了 docker desktop 的话，可以不需要下载 `hyperkit` 或者 `virtualbox`。

```bash
# Install hyperkit and minikube (check which vm-driver to use, if install docker desktop already, you can just use vm-driver=docker instead of install hyperkiy)
brew install hyperkit
brew install minikube
```

```bash
# Start minikube 
minikube start --vm-driver hyperkit --container-runtime=docker
# minikube start --vm-driver virtualbox --container-runtime=docker
# minikube start --vm-driver docker --container-runtime=docker

# Tell Docker CLI to talk to minikube's VM
eval $(minikube docker-env)

# Save IP to a hostname
echo "`minikube ip` docker.local" | sudo tee -a /etc/hosts > /dev/null

# Test
docker run hello-world
```

> 注意：如果本地已经安装了 docker desktop，那么可以使用 minikube start --vm-driver docker --container-runtime=docker 来快速启动 minikube

**minikube Cheatsheet**

`minikube stop` 不会删除任何数据，只是停止 VM 和 k8s 集群。

`minikube delete` 删除所有 minikube 启动后的数据。

`minikube ip` 集群和 docker enginer 运行的 IP 地址。

`minikube pause` 暂停当前的资源和集群

`minikube status` 查看当前集群状态

### 安装 k8s CLI 和 Terminal based UI

如果本地没有k8s CLI `kubectl` 的话，需要安装一下

```bash
brew install kubectl
```

如果我们希望更直观的观察 kubenates 中资源的变化，也可以安装一个 [k9s](https://k9scli.io/)（对于初学者而言，更建议使用 kubectl 来手动观察）

```bash
brew install k9s
```

如果需要练习 helm 的使用，可以先安装 helm

```bash
brew install helm
```

### 注册 docker hub 账号登录

在 docker hun(https://hub.docker.com/) 中注册账号，并且使用 login 登录账号。

```
docker login
```

## Container

```ruby
# app.rb
require "sinatra" // 类似 java 的 import

set :bind, "0.0.0.0"

get "*" do
  "[v1] Hello, Kubernetes!\n"
end
```

```dockerfile
# Dockerfile
FROM ruby:2.6.1-alpine3.9

RUN apk add curl && gem install sinatra
COPY app.rb .
ENTRYPOINT ["ruby", "app.rb"]
```

```
docker build . -t guangzhengli/hellok8s:v1 // build 一个镜像到本地, 镜像名称为 guangzhengli/hellok8s:v1
```

```
docker run -p 4567:4567 --name hellok8s -d guangzhengli/hellok8s:v1 // 启动一个 container, 设置端口映射、名称
```

```
docker push guangzhengli/hellok8s:v1 // 将该镜像 push 到 https://hub.docker.com/, 因为我们刚刚 docker login 过
```

## Pod

```yaml
# nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx // 自定义 pod 名称
spec:
  containers:
    - name: nginx-container // 自定义 container 名称
      image: nginx // docker 官网上 nginx 镜像的名称, 未指定版本号, 故拉的最新版本
```

```shell
kubectl apply -f nginx.yaml // 通过 nginx.yaml 文件启动对应的资源, 此处只有一个 pod 资源

kubectl port-forward nginx 3001:80 // 将容器内的 80 端口映射到本地 3001 端口

kubectl logs --follow nginx // 查看日志, 对应 k9s 的 l 命令
               ^        ^
               |        |
                ------------- tells kubectl to keep
                        |     streaming the logs live,
                        |     instead of just
                        |     printing and exiting
                        |
                         ---- the name of the pod we want
                              to see the logs
                              
kubectl exec nginx -- ls

root@nginx:/ echo "it works!" > /usr/share/nginx/html/index.html

kubectl delete pod nginx
# pod "nginx" deleted

kubectl delete -f nginx.yaml
# pod "nginx" deleted
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: httpd
spec:
  containers:
    - name: httpd-container
      image: registry.hub.docker.com/library/httpd

# default page in: /usr/local/apache2/htdocs/index.html
```

```yaml
# hellok8s.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hellok8s
spec:
  containers:
    - name: hellok8s-container
      image: guangzhengli/hellok8s:v1
```

```
kubectl port-forward hellok8s 4567:4567
```

## Deployment

> deployment 执行后, 生成的 pod 名称会有3段, 例如 hellok8s-6477bb6d56-8hvqw, 多个 pod 前两段名称是一样的, 第三个不一样, 前两段名称就是 replicaset 的名称, 对 deployment 做变更后(例如版本升级), 会新生成一个 replicaset
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s // 自定义 deployment 名称
spec:
  replicas: 1 // 副本数量
  selector:
    matchLabels:
      app: hellok8s // 需要与 pod 的 metadata.name 一致, 否则无法管理
  template: // template 就是指 pod 信息
    metadata:
      labels:
        app: hellok8s // 对应单独配置 pod 资源时的 metadata.name
    spec:
      containers:
        - image: guangzhengli/hellok8s:v1 // 版本控制
          name: hellok8s-container
```

```shell
kubectl apply -f deployment.yaml

kubectl get pods       
# NAME                        READY   STATUS    RESTARTS
# hellok8s-6678f66cb8-42jtr   1/1     Running   0       


# Replace the pod name with your local pod name
kubectl delete pod hellok8s-6678f66cb8-42jtr
# pod "hellok8s-6678f66cb8-42jtr" deleted

kubectl get pods
# NAME                        READY   STATUS    RESTARTS  
# hellok8s-6678f66cb8-8nqf2   1/1     Running   0
```

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s
spec:
  replicas: 10
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
      - image: guangzhengli/hellok8s:v1
        name: hellok8s-container
```

### release new verison

```ruby
get "*" do
  "[v2] Hello, Kubernetes!\n"
end
```

```shell
docker build . -t guangzhengli/hellok8s:v2
docker push guangzhengli/hellok8s:v2
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
      - image: guangzhengli/hellok8s:v2
        name: hellok8s-container
```

```shell
kubectl get pods
# NAME                        READY   STATUS
# hellok8s-6678f66cb8-52zt9   1/1     Running
# hellok8s-6678f66cb8-nxphs   1/1     Running

# You can locally run the port forward command by replacing the pod name
kubectl port-forward hellok8s-6678f66cb8-52zt9 4567:4567

# or open another terminal and run the followig command
curl http://localhost:4567
# [v2] Hello, Kubernetes!
```

### Rolling Update

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s
spec:
  strategy:
     rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  replicas: 3
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
      - image: guangzhengli/hellok8s:v2
        name: hellok8s-container
```

```ruby
require "sinatra"

set :bind, "0.0.0.0"

$counter = 0

get "*" do
  $counter += 1
  if $counter > 3
    raise "Whoops, something is wrong"
  end

  "[bad] Hello, Kubernetes!\n"
end
```

```
docker build . -t guangzhengli/hellok8s:bad
docker push guangzhengli/hellok8s:bad
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
      - image: guangzhengli/hellok8s:bad
        name: hellok8s-container
```

```shell
kubectl rollout history deployment hellok8s
kubectl rollout undo deployment hellok8s
kubectl rollout undo deployment/hellok8s --to-revision=2
```

### Automatically blocking bad releases by readinessProbe

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
      - image: guangzhengli/hellok8s:v2 # Still using v2
        name: hellok8s-container
        readinessProbe: # New readiness probe // 探活配置, 如果探活失败, 则当前 pod 会失败, 且不会重启其他 pod
          periodSeconds: 1
          successThreshold: 5
          httpGet:
            path: /
            port: 4567
```

```shell
kubectl describe pod hellok8s-68f47f657c-zwn6g

# ...
# ...
# ...
# Readiness probe failed:
# HTTP probe failed with statuscode: 500
```

## Service

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: hellok8s-svc
spec:
  type: NodePort
  selector:
    app: hellok8s // 对应 pod 的 name
  ports:
  - port: 4567 // pod 的端口号
    nodePort: 30001 // service 暴露出来的端口号
```

```
kubectl apply -f service.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
      - image: guangzhengli/hellok8s:v3
        name: hellok8s-container
```

```
kubectl get service hellok8s-svc
```

```ruby
#app.rb
require "sinatra"

set :bind, "0.0.0.0"

get "*" do
  "[v3] Hello, Kubernetes, from #{`hostname`.strip}!\n"
end
```

```
docker build . -t guangzhengli/hellok8s:v3
docker push guangzhengli/hellok8s:v3
```

```shell
kubectl apply -f deployment.yaml

curl http://localhost:30001
# [v3] Hello, Kubernetes, from hellok8s-7f4c57d446-t9ngx!

curl http://localhost:30001
# [v3] Hello, Kubernetes, from hellok8s-7f4c57d446-985dq!

curl http://localhost:30001
# [v3] Hello, Kubernetes, from hellok8s-7f4c57d446-t9ngx!
```

## ingress

> 定义路由, 和 nginx 路由作用类似

```shell
minikube addons enable ingress

kubectl delete deployment,service --all
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hellok8s-svc
spec:
  selector:
    app: hellok8s
  ports:
  - port: 4567
    targetPort: 4567

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
      - image: guangzhengli/hellok8s:v3
        name: hellok8s-container
```

```yaml
# nginx.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
  - port: 1234
    targetPort: 80

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx-container
```

```shell
kubectl apply -f hellok8s.yaml
# service/hellok8s-svc created
# deployment.apps/hellok8s created

kubectl apply -f nginx.yaml
# service/nginx-svc created
# deployment.apps/nginx created

kubectl get pods
# NAME                        READY   STATUS    RESTARTS 
# hellok8s-7f4c57d446-6c8b8   1/1     Running   0        
# hellok8s-7f4c57d446-jkqbl   1/1     Running   0        
# nginx-77c5c66899-dgkk2      1/1     Running   0        
# nginx-77c5c66899-w9srw      1/1     Running   0        

kubectl get service
# NAME           TYPE        CLUSTER-IP       PORT(S)   
# hellok8s-svc   ClusterIP   10.102.242.233   4567/TCP  
# nginx-svc      ClusterIP   10.96.19.78      1234/TCP
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-ingress
  annotations:
    # We are defining this annotation to prevent nginx
    # from redirecting requests to `https` for now
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
    - http:
        paths:
          - path: / // "/"请求打到 nginx-svc 这个 service 资源上
            pathType: Prefix
            backend:
              service:
                name: nginx-svc
                port:
                  number: 1234
          - path: /hello // "/hello"请求打到 hellok8s-svc 这个 service 资源上
            pathType: Prefix
            backend:
              service:
                name: hellok8s-svc
                port:
                  number: 4567
```

```shell
kubectl apply -f ingress.yaml
# ingress.extensions/hello-ingress created

kubectl get ingress
# NAME            HOSTS   ADDRESS     PORTS   AGE
# hello-ingress   *       localhost   80      1m
```

```shell
kubectl apply -f ingress.yaml
# ingress.extensions/hello-ingress configured

curl http://localhost/hello
# [v3] Hello, Kubernetes, from hellok8s-7f4c57d446-qth54!

curl http://localhost
# (nginx welcome page)
```

## Configmap

### env var

```ruby
require "sinatra"

set :bind, "0.0.0.0"

get "*" do
  message = ENV.fetch("MESSAGE", "Hello, Kubernetes")
  "[v4] #{message} (from #{`hostname`.strip})\n"
end
```

```
docker build . -t guangzhengli/hellok8s:v4
docker push guangzhengli/hellok8s:v4

kubectl delete deployment,service,ingress --all
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hellok8s-svc
spec:
  type: NodePort
  selector:
    app: hellok8s
  ports:
  - port: 4567
    nodePort: 30001

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
      - image: guangzhengli/hellok8s:v4
        name: hellok8s-container
```

```shell
kubectl apply -f hellok8s.yaml 

curl localhost:30001
# [v4] Hello, Kubernetes (from hellok8s-69dbd44879-vt8dv)
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
      - image: guangzhengli/hellok8s:v4
        name: hellok8s-container
        env:
          - name: MESSAGE
            value: "It works!"
```

```shell
kubectl apply -f hellok8s.yaml 

curl localhost:30001
# [v4] It works! (from hellok8s-568f64dd94-bfxhs)
```

### configmap

```yaml
# hellok8s-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hellok8s-config
data:
  MESSAGE: "It works with a ConfigMap!"
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
      - image: guangzhengli/hellok8s:v4
        name: hellok8s-container
        env:
          - name: MESSAGE
            valueFrom:
              configMapKeyRef:
                name: hellok8s-config
                key: MESSAGE
```

```shell
kubectl apply -f hellok8s-config.yaml

kubectl apply -f hellok8s.yaml

kubectl apply -f service.yaml

curl localhost:30001
# [v4] It works with a ConfigMap! (from hellok8s-54d5fb5765-nl62z)
```

### Getting all the variables from a ConfigMap

```
env:
  - name: VAR1
    valueFrom:
      configMapKeyRef:
        name: hellok8s-config
        key: VAR1

  - name: VAR2
    valueFrom:
      configMapKeyRef:
        name: hellok8s-config
        key: VAR2

  - name: VAR3
    valueFrom:
      configMapKeyRef:
        name: hellok8s-config
        key: VAR3
# ...
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
      - image: guangzhengli/hellok8s:v4
        name: hellok8s-container
        envFrom:
          - configMapRef:
              name: hellok8s-config
```

```
kubectl apply -f hellok8s-updated.yaml

curl localhost:30001
# [v4] It works with a ConfigMap! (from hellok8s-54d5fb5765-nl62z)
```

### Exposing ConfigMap as files

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      volumes:
       - name: config
         configMap:
           name: hellok8s-config
      containers:
      - image: guangzhengli/hellok8s:v4
        name: hellok8s-container
        volumeMounts:
        - name: config
          mountPath: /config
```

```shell
kubectl apply -f hellok8s.yaml
kubectl apply -f hellok8s-config.yaml

kubectl get pods
# NAME                       READY   STATUS
# hellok8s-8c56675c9-7gxpv   1/1     Running
# hellok8s-8c56675c9-bfk8t   1/1     Running

# Replace the pod name to what you have running locally
kubectl exec -it hellok8s-8c56675c9-7gxpv -- sh
cat /config/MESSAGE
# It works with a ConfigMap!
```

## Secret

```yaml
# hellok8s-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: hellok8s-secret
data:
  SECRET_MESSAGE: "SXQgd29ya3Mgd2l0aCBhIFNlY3JldAo="
```

```
echo 'It works with a Secret' | base64
# SXQgd29ya3Mgd2l0aCBhIFNlY3JldAo=
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hellok8s
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hellok8s
  template:
    metadata:
      labels:
        app: hellok8s
    spec:
      containers:
      - image: guangzhengli/hellok8s:v4
        name: hellok8s-container
        env:
          - name: MESSAGE
            valueFrom:
              secretKeyRef:
                name: hellok8s-secret
                key: SECRET_MESSAGE
```

```shell
kubectl apply -f hellok8s-secret.yaml
kubectl apply -f deployment.yaml

kubectl get pods
# NAME                        READY   STATUS
# hellok8s-6d7579848d-f56wb   1/1     Running
# hellok8s-6d7579848d-kzq57   1/1     Running

# Replace the pod name to what you have running locally
kubectl exec -it hellok8s-6d7579848d-kzq57 --  env | grep MESSAGE
# MESSAGE=It works with a Secret
```

### Using stringData

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: hellok8s-secret
stringData:
  SECRET_MESSAGE: "It works with a Secret" // 自动 base64, 读取数据后需要 decode
```

```shell
kubectl get secret hellok8s-secret -o yaml

# apiVersion: v1
# kind: Secret
# data:
#   SECRET_MESSAGE: SXQgd29ya3Mgd2l0aCBhIFNlY3JldAo=
# ...
```

## helm

```shell
helm create helm-hello

helm upgrade --install hello-helm --values values.yaml .

helm list

helm rollback hello-helm
```



