---
title: "Tkeel部署"
draft: false
toc: true
description: ""
featured_image:
categories: []
tags: [IOT, k8s]
---
Tkeel 云原生物联网平台初探<!--more-->
#### 1 install
##### 1.1 k8s手动安装可能遇到的问题
1. 虚拟机网卡启动失败  
   `network-manager服务与network.service冲突；关闭network-manage服务并在systemd服务中禁用掉`
2. 安装速度太慢  
   ` docker 设置国内镜像源，或者使用代理，将需要的镜像提前下到本地`

#### 2 docker
##### 2.1 images
- docker images
    ```
    # output:
    REPOSITORY                           TAG                  IMAGE ID       CREATED         SIZE
    tkeelio/rudder                       dev20220513          9e9725a9e275   3 days ago      61.8MB
    tkeelio/keel                         dev20220513
    ```
- docker rmi {IMAGE ID}
##### 2.2 container
- docker ps [-a]
    ```
    # output:
    CONTAINER ID   IMAGE                  COMMAND                  CREATED      STATUS                PORTS                                                NAMES
    71b3272a6e55   openzipkin/zipkin      "start-zipkin"           5 days ago   Up 5 days (healthy)   9410/tcp, 0.0.0.0:9411->9411/tcp, :::9411->9411/tcp  dapr_zipkin
    9da93349655f   daprio/dapr:1.7.1      "./placement"            5 days ago   Up 5 days             0.0.0.0:50005->50005/tcp, :::50005->50005/tcp        dapr_placement
    0603d14ee8f8   redis                  "docker-entrypoint.s…"   5 days ago   Up 5 days             0.0.0.0:6379->6379/tcp, :::6379->6379/tcp
    ```
- docker rm {CONTAINER ID}
##### 2.3 volume
##### 2.4 build
1. Dockerfile
   ```
   FROM alpine:3.13
   ENV PLUGIN_ID=rudder
   COPY ./rudder /
   CMD ["/rudder"]
   ```
2. docker build  
   **docker build -t {IMAGE}:{IMAGE TAG} -f {Dockerfile} {Workspace}**
   ```
   # 在当前目录 ./ 下通过目录下的 Dockerfile 构建 tag 为 test/nginx:dev-20220516 的镜像到本地
   # 构建完成后在本地可以通过 $ docker images 查看
   $ docker build -t test/nginx:dev-20220516 -f ./Dockerfile .
   ```
##### 2.5 containerd
```
$ ctr -n k8s.io images ls
$ ctr -n k8s.io i import *.tar
$ ctr -n k8s.io i rm ImageID
```

#### 3 k8s
##### 3.1 k8s集群访问（kubectl）
###### 3.1.1 配置文件
1. 配置文件位置
    ```
    # 用来访问集群的配置文件config位置
    # 自行搭建的集群: /etc/kubernetes/admin.conf or ~/.kube/conf
    # 用其他方式搭建的集群: ~/.kube/conf
    $ kubectl config view
    # output:
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority-data: DATA+OMITTED
        server: https://127.0.0.1:42177          
      name: kind-kind
    contexts:
    - context:
        cluster: kind-kind
        user: kind-kind
      name: kind-kind
    current-context: kind-kind
    kind: Config
    preferences: {}
    users:
    - name: kind-kind
      user:
        client-certificate-data: REDACTED
        client-key-data: REDACTED
    ```
2. 指定配置文件
    ```
    $ kubectl --kubeconfg=~/.kube/config-other cluster-info
    ```
3. 设置环境变量（推荐使用）
    ```
    # 推荐写入到 ~/.zshrc or ~/.bashrc
    # kubectl 会自动将KUBECONFIG环境变量中的多个有效配置合并
    $ export KUBECONFIG=~/.kube/config:~/.kube/config-123.9:~/.kube/config-qingcloud02:~/.kube/config-k5:~/.kube/config-home-vm02
    # 激活环境变量
    $ source ~/.zshrc or source ~/.bashrc  
    # 后续可以直接使用kubectl
    $ kubectl cluster-info
    ```
###### 3.1.2 多集群配置管理
   - 多集群配置切换
     ```
     # 列出当前配置文件中可用的集群配置上下文
     # 带 * 的代表当前使用的集群上下文
     $ kubectl config get-contexts 
     # output:
     CURRENT   NAME                          CLUSTER          AUTHINFO           NAMESPACE
               docker-desktop                docker-desktop   docker-desktop
               kind-app01                    kind-app01       kind-app01
     *         kind-kind                     kind-kind        kind-kind
               kubernetes-admin@kubernetes   kubernetes       kubernetes-admin
     # 切换上下文
     $ kubectl config use-context kubernetes-admin@kubernetes
     # output:
     Switched to context "kubernetes-admin@kubernetes".
     # 查看集群信息,获取控制节点信息
     $ kubectl cluster-info
     # output:
     Kubernetes control plane is running at https://127.0.0.1:42177
     CoreDNS is running at https://127.0.0.1:42177/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
     # 获取集群所有节点信息
     $ kubectl get nodes
     # 单节点的k8s集群
     # output:
     NAME                 STATUS   ROLES                  AGE    VERSION
     kind-control-plane   Ready    control-plane,master   7d3h   v1.23.0
     # 获取集群节点详细信息
     $ kubectl get nodes -o wide
     # 多节点的k8s集群
     # output:
     NAME          STATUS   ROLES    AGE    VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
     master1       Ready    master   232d   v1.19.8   192.168.123.9    <none>        Ubuntu 18.04.5 LTS   4.15.0-121-generic   docker://20.10.6
     worker-s001   Ready    worker   232d   v1.19.8   192.168.123.11   <none>        Ubuntu 18.04.5 LTS   4.15.0-121-generic   docker://20.10.6
     worker-s002   Ready    worker   232d   v1.19.8   192.168.123.12   <none>        Ubuntu 18.04.5 LTS   4.15.0-121-generic   docker://20.10.6
     worker-s003   Ready    worker   89d    v1.19.8   192.168.123.3    <none>        Ubuntu 18.04.5 LTS   4.15.0-121-generic   docker://20.10.6
     ```

###### 3.1.3 多集群配置冲突
- 多集群默认配置中user/cluster/context使用kubernetes-admin/kubernetes/kubernetes-admin@kubernetes导致kubectl config切换失败
```
# 对每个配置文件中的user/cluster/context重命名使用不同的名称
```
- other

##### 3.2 命名空间
1. 基本概念: Namespace是对一组资源和对象的抽象集合，比如可以用来将系统内部的对象划分为不同的项目组或用户组。
2. 查看命名空间
    ```
    $ kubectl get namespaces
    # output:
    NAME                              STATUS   AGE
    default                           Active   7d4h
    kube-node-lease                   Active   7d4h
    kube-public                       Active   7d4h
    kube-system                       Active   7d4h
    kubesphere-controls-system        Active   2d23h
    kubesphere-monitoring-federated   Active   2d23h
    kubesphere-monitoring-system      Active   2d23h
    kubesphere-system                 Active   2d23h
    local-path-storage                Active   7d4h
    testing                           Active   7d4h
    ```
3. 查看某个命名空间下的资源(不加命名空间默认找default下面的)
    ```
    # 查看所有命名空间下的pod
    $ kubectl get pods --all-namespaces
    # output:
    NAMESPACE                      NAME                                                  READY   STATUS    RESTARTS           AGE
    kube-system                    coredns-64897985d-cm7ct                               1/1     Running   0                  7d5h
    kube-system                    coredns-64897985d-hvsvx                               1/1     Running
    testing                        clickhouse-tkeel-core-s0-r0-0                         1/1     Running   0                  7d4h
    testing                        console-plugin-admin-custom-config-78486bc8d8-zqr6l   2/2     Running
    # 查看testing命名空间下的pod
    $ kubectl get pods -n testing
    # output:
    NAMESPACE                      NAME 
    testing                        clickhouse-tkeel-core-s0-r0-0                          1/1     Running   0                 7d4h
    testing                        console-plugin-admin-custom-config-78486bc8d8-zqr6l    2/2     Running
    ```
##### 3.3 内置资源
###### 3.3.1 pod
1. 基本概念: 基本调度单元，多个容器的集合
    ```
    $ kubectl explain pod
    # output:
    KIND:     Pod
    VERSION:  v1
    
    DESCRIPTION:
         Pod is a collection of containers that can run on a host. This resource is
         created by clients and scheduled onto hosts.
    
    FIELDS:
       apiVersion   <string>  # schema API版本
       kind <string>          # 资源类型
       metadata     <Object>  # 定义的元信息
       spec <Object>          # 详细的配置描述
       status       <Object>  # 运行状态
    ```
2. 查看pod信息
    ```
    # 注意查看资源信息时必须加命名空间
    $ kubectl get pod -n testing
    # output:
    NAME                                                              READY   STATUS             RESTARTS         AGE
    clickhouse-tkeel-core-s0-r0-0                                     1/1     Running            0                7d4h
    console-plugin-admin-custom-config-78486bc8d8-zqr6l               2/2     Running            0                6d
    console-plugin-admin-plugins-9c85d4564-6j6bv                      2/2     Running            0                7d3h
    $ kubectl describe pod clickhouse-tkeel-core-s0-r0-0  -n testing
    # output:
    Name:         clickhouse-tkeel-core-s0-r0-0
    Namespace:    testing
    Priority:     0
    Node:         kind-control-plane/172.18.0.2
    Start Time:   Mon, 09 May 2022 11:41:45 +0800
    Labels:       app.kubernetes.io/instance=tkeel-core
    Annotations:  clickhouse/config-checksum: 1029ef1cb57f6f09e68e54c33ff1e99b66975a2fae83015f43e0839c23383e70
    Status:       Running
    Controlled By:  StatefulSet/clickhouse-tkeel-core-s0-r0
    Containers:
      clickhouse:
        Container ID:   containerd://54d2ebbdd918f2e71214d7fd0ab2f69adab1abd5f381255ff493e6aa41422eb9
        Image:          yandex/clickhouse-server:22.1.3.7
        Image ID:       docker.io/yandex/clickhouse-server@sha256:1cbf75aabe1e2cc9f62d1d9929c318a59ae552e2700e201db985b92a9bcabc6e
        Ports:          9000/TCP, 8123/TCP
        Host Ports:     0/TCP, 0/TCP
        State:          Running
          Started:      Mon, 09 May 2022 11:45:53 +0800
        Ready:          True
    ```
3. 基础配置
    ```
    apiVersion: v1
    kind: Pod
    metadata:
      name: redis
    spec:
      containers:
      - name: redis
        image: redis
        volumeMounts:
        - name: redis-storage
          mountPath: /data/redis
      volumes:
      - name: redis-storage
        emptyDir: {}
    ```
4.字段解析
```
1. Volume
2. RestartPoliy
- Always：只要退出就重启
- OnFailure：失败退出（exit code不等于0）时重启
- Never：只要退出就不再重启
3. env-环境变量
- hostname
- 容器和Pod的基本信息
- 集群中服务的信息
4. ImagePullPolicy
- Always：不管镜像是否存在都会进行一次拉取
- Never：不管镜像是否存在都不会进行拉取 
- IfNotPresent：只有镜像不存在时，才会进行镜像拉取，默认为IfNotPresent
5. resources
- spec.containers[].resources.limits.cpu：CPU上限，可以短暂超过，容器也不会被停止 
- spec.containers[].resources.limits.memory：内存上限，不可以超过；如果超过，容器可能会被停止或调度到其他资源充足的机器上
- spec.containers[].resources.requests.cpu：CPU请求，可以超过
- spec.containers[].resources.requests.memory：内存请求，可以超过；但如果超过，容器可能会在Node内存不足时清理
6. 探针
# 两种探针支持exec、tcp和httpGet方式三种方式探测容器状态
- LivenessProbe：探测应用是否处于健康状态，如果不健康则删除重建改容器
- ReadinessProbe：探测应用是否启动完成并且处于正常服务状态，如果不正常则更新容器的状态
7. init container
# Init Container在所有容器运行之前执行
8. 生命周期&钩子
- postStart： 容器启动后执行，注意由于是异步执行，它无法保证一定在ENTRYPOINT之后运行。如果失败，容器会被杀死，并根据RestartPolicy决定是否重启
- preStop：容器停止前执行，常用于资源清理。如果失败，容器同样也会被杀死
# 钩子的回调函数支持两种方式：
# exec：在容器内执行命令
# httpGet：向指定URL发起GET请求
9. nodeSelector
# 指定该Pod只想运行在带有id=node1标签的Node上
11. todo
```
###### 3.3.2 service
1. 基本概念
   ```
   # 4层负载均衡TCP/UDP
   # CluesterIP && ExternalIP
   # Label Selector
   # Headless Services
   # External Name
   # IPVS
   # endpoints
   ```
- ClusterIP
- NodePort
- LoadBalancer
2. 查看service信息
   ```
   $ kubectl get svc -n testing
   $ kubectl describe svc/app-service -n testing
   
   # DNS
   # _my-port-name._my-port-protocol.my-svc.my-namespace.svc.cluster-domain.example
   # my-svc.my-namespace.svc.cluster-domain.example
   ~ $ cat /etc/resolv.conf
   # output:
   # search ps.svc.cluster.local svc.cluster.local cluster.local ap2a.qingcloud.com
   # nameserver 10.96.0.10
   # options ndots:5
   
   # endpoints
   $ kubectl get endpoints -n testing
   ```
3. 基础配置
    ```
    kind: Service
    apiVersion: v1
    metadata:
      name: my-service
    spec:
      selector:
        app: MyApp
      ports:
        - protocol: TCP
          port: 80
          targetPort: 9376
    ```
- 1
- 2
###### 3.3.3 deployment
1. 基础概念
   ```
   用来管理pod，实现滚动升级和回滚以及应用的扩容和缩容
   ```
2. 查看deployment信息
   ```
   $ kubectl get deploy -n testing
   $ kubeclt get deploy/nginx -n testing -o wide
   $ kubeclt get deploy/nginx -n testing -o yaml
   $ kubeclt describe deploy/nginx -n testing
   ```
3. 基础配置
    ```
    apiVersion: extensions/v1beta1
    kind: Deployment
    metadata:
      name: nginx-deployment
    spec:
      replicas: 3
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx:1.7.9
            ports:
            - containerPort: 80   
    ```
- 1 升级&回滚
  ```
  # 查看更新历史
  $ kubectl rollout history deployment/nginx-deployment
  # 回退到上一个版本
  $ kubectl rollout undo deployment/nginx-deployment
  # 回退到历史指定版本
  $ kubectl rollout undo deployment/nginx-deployment --to-revision=1
  # 通过设置.spec.revisonHistoryLimit项来指定deployment最多保留多少revison历史记录。默认的会保留所有的revision；如果将该项设置为0，Deployment就不允许回退了
  ```
- 2 pod 扩容
  ```
  # 调整pod的副本数
  $ kubectl scale deployment nginx-deployment --replicas 10
  # 基于HPA的自动扩容，基于当前系统资源自动选择合适的副本数量
  $ kubectl autoscale deployment nginx-deployment --min=3 --max=10 --cpu-percent=80
  ```
###### 3.3.4 ingress && ingress-controller
1. 基础概念
2. 查看ingress信息
3. 基础配置
- 1
- 2
###### 3.3.5 pv && pvc
1. 基础概念
2. 查看ingress&&ingress-controller信息
3. 基础配置
- 1
- 2

#### 4 helm
##### 4.1 生命周期
1. 用户返回 helm install foo
2. Helm库调用安装API
3. 在 crds/目录中的CRD会被安装
4. 在一些验证之后，库会渲染foo模板
5. 库准备执行pre-install钩子(将hook资源加载到Kubernetes中)
6. 库按照权重对钩子排序(默认将权重指定为0)，然后在资源种类排序，最后按名称正序排列
7. 库先加载最小权重的钩子(从负到正)
8. 库会等到钩子是 "Ready"状态(CRD除外)
9. 库将生成的资源加载到Kubernetes中。注意如果设置了--wait参数，库会等所有资源是ready状态， 且所有资源准备就绪后才会执行post-install钩子。
10. 库执行post-install钩子(加载钩子资源)。
11. 库会等到钩子是"Ready"状态
12. 库会返回发布对象(和其他数据)给客户端
13. 客户端退出
##### 4.2 Values
value值的优先级 由高到低
- 使用--set (比如helm install --set foo=bar ./mychart)传递的单个参数
- 使用-f参数(helm install -f myvals.yaml ./mychart)传递到 helm install 或 helm upgrade的values文件
- 如果是子chart，就是父chart中的values.yaml文件
- chart中的values.yaml文件
##### 4.3 常用操作
```
# 安装自定义的chart
$ helm install full-coral ./test_chart/

# 获取安装之后的chart清单
$ helm get manifest full-coral

# 卸载chart清单相关的资源
$ helm uninstall full-cora

# 展示渲染之后的模版，只渲染不安装
$ helm install --debug --dry-run goodly-guppy ./test_chart/
# output: 
install.go:178: [debug] Original chart version: ""
install.go:195: [debug] CHART PATH: /root/tmp/helm_test/test_chart

NAME: goodly-guppy
LAST DEPLOYED: Fri May 20 16:35:27 2022
NAMESPACE: default
STATUS: pending-install
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
{}

COMPUTED VALUES:
{}

HOOKS:
MANIFEST:
---
# Source: test_chart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: goodly-guppy-configmap
data:
  myvalue: "Hello World"
```
##### 4.4 helm插件开发
###### 4.4.1 步骤
[插件脚手架](https://docs.tkeel.io/developer_cookbook/tkeel_plugin/create)
###### 4.4.2 注意事项
1. 插件注解
- tkeel插件注解
```
# 在tkeel中插件分为系统插件和自定义插件
# 自定义的插件需要在chart.yaml中按以下格式加入tkeel注解，否则在插件列表中默认会被系统过滤掉
- annotations:
   {
   tkeel.io/deployment-name: prometheus,
   tkeel.io/enable: "true",
   tkeel.io/plugin-port: "9202",
   tkeel.io/tag: prometheus,
   tkeel.io/version: v0.0.1
   }
```
- dapr应用注解
[dapr配置文件](https://docs.dapr.io/operations/configuration/invoke-allowlist/)
```
# template.metadata.annotations 中需要启用dapr相关配置
annotations:
  {
    "dapr.io/enabled": "true",
    "dapr.io/app-id": prometheus,
    "dapr.io/app-port": "9202",
    "dapr.io/dapr-grpc-port": "50001",
    "dapr.io/dapr-http-port": "3500",
    "dapr.io/metrics-port": "9089",
    "dapr.io/log-level": debug,
    "dapr.io/enable-api-logging": "true"
  }

# 在插件pod启动后，会在annotations中插入dapr配置文件 dapr.io/config: prometheus
annotations:
    dapr.io/app-id: prometheus
    dapr.io/app-port: '9202'
    dapr.io/config: prometheus
    dapr.io/dapr-grpc-port: '50001'
    dapr.io/dapr-http-port: '3500'
    dapr.io/enable-api-logging: 'true'
    dapr.io/enabled: 'true'
    dapr.io/log-level: debug
    dapr.io/metrics-port: '9089'
    kubesphere.io/restartedAt: '2022-06-06T08:33:19.415Z'

# 该配置文件为自定义的CRD类型，可以在启动配置请求的访问权限/链路追踪的插件等等，详情见上述官网链接 
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  annotations:
    meta.helm.sh/release-name: prometheus
    meta.helm.sh/release-namespace: testing
  labels:
    app.kubernetes.io/managed-by: Helm
  name: prometheus
  namespace: testing
spec:
  accessControl:
    defaultAction: deny
    policies:
      - appId: rudder
        defaultAction: allow
        namespace: testing
        trustDomain: tkeel
      - appId: keel
        defaultAction: allow
        namespace: testing
        trustDomain: tkeel
      - appId: rudder
        defaultAction: allow
        namespace: testing
        trustDomain: public   # 本地测试时设置为 public 避免 rudder 调用插件接口时 domain 不匹配 tkeel 导致 permission access deny
    trustDomain: tkeel
  httpPipeline:
    handlers:
      - name: prometheus-oauth2-client
        type: middleware.http.oauth2clientcredentials
  metric:
    enabled: true
```
2. other
#### 5 middleware
##### 5.1 redis
###### 5.1.1 install
```
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm repo update
$ helm install redis bitnami/redis
# output: 
NAME: redis
LAST DEPLOYED: Thu May 19 14:58:47 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: redis
CHART VERSION: 16.9.6
APP VERSION: 6.2.7

** Please be patient while the chart is being deployed **

Redis&trade; can be accessed on the following DNS names from within your cluster:

    redis-master.default.svc.cluster.local for read/write operations (port 6379)
    redis-replicas.default.svc.cluster.local for read-only operations (port 6379)



To get your password run:

    export REDIS_PASSWORD=$(kubectl get secret --namespace default redis -o jsonpath="{.data.redis-password}" | base64 --decode)

To connect to your Redis&trade; server:

1. Run a Redis&trade; pod that you can use as a client:

   kubectl run --namespace default redis-client --restart='Never'  --env REDIS_PASSWORD=$REDIS_PASSWORD  --image docker.io/bitnami/redis:6.2.7-debian-10-r19 --command -- sleep infinity

   Use the following command to attach to the pod:

   kubectl exec --tty -i redis-client --namespace default -- bash

2. Connect using the Redis&trade; CLI:
   REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis-master
   REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis-replicas

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace default svc/redis-master 6379:6379 &
    REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h 127.0.0.1 -p 6379
```
###### 5.1.2 access
```
$ kubectl port-forward svc/tkeel-middleware-redis-master 6379:6379 -n testing
```

#### 6 tkeel
##### 6.1 install
- logs
```
$ kubectl get pods -n keel-system | grep rudder | awk '{print $1}' | head -n 1 | xargs -I {}  kubectl  logs {} -n keel-system --tail=10
$ kubectl get pods -n keel-system | grep keel | awk '{print $1}' | head -n 1 | xargs -I {}  kubectl logs {} -n keel-system --tail=10
```
##### 6.2 遇到的问题
- 插件安装失败
```
→  Deploying the tKeel Platform to your cluster...
↓  Deploying the tKeel Platform to your cluster... ℹ️  install tKeel middleware done.
↗  Deploying the tKeel Platform to your cluster... ℹ️  install tKeel platform  done.
→  Deploying the tKeel Platform to your cluster... W0509 11:41:40.516252   13571 warnings.go:70] policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
W0509 11:41:40.556146   13571 warnings.go:70] policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
↙  Deploying the tKeel Platform to your cluster... ℹ️  install tKeel component <core> done.
↑  Deploying the tKeel Platform to your cluster... ℹ️  install tKeel component <rudder> done.
✅  Deploying the tKeel Platform to your cluster...
http://127.0.0.1:39083/v1.0/invoke/rudder/method/v1/oauth2/admin?password=Y2hhbmdlbWU%3D
http://127.0.0.1:35137/v1.0/invoke/keel/method/apis/rudder/v1/repos/tkeel
http://127.0.0.1:43809/v1.0/invoke/keel/method/apis/rudder/v1/plugins/console-portal-admin
❌  Install "console-portal-admin" failed, Because: can't handle this
http://127.0.0.1:33349/v1.0/invoke/keel/method/apis/rudder/v1/plugins/console-portal-tenant
❌  Install "console-portal-tenant" failed, Because: can't handle this
http://127.0.0.1:43631/v1.0/invoke/keel/method/apis/rudder/v1/plugins/console-plugin-admin-plugins
❌  Install "console-plugin-admin-plugins" failed, Because: can't handle this
✅  Success! tKeel Platform has been installed to namespace testing. To verify, run `tkeel plugin list' in your terminal. To get started, go here: https://tkeel.io/keel-getting-started
```
```
# 解决方式
$ root@i-onaxtmf3:~# tkeel admin login -p changeme --print
# output:
http://127.0.0.1:37653/v1.0/invoke/rudder/method/v1/oauth2/admin?password=Y2hhbmdlbWU%3D
✅  You are Login Success!
✅  Your Token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJ0S2VlbCIsImV4cCI6IjIwMjItMDUtMDlUMDY6MjE6NTYuNDQwNDgwNDE4WiIsImlhdCI6IjIwMjItMDUtMDlUMDU6MjE6NTYuNDQwNDgwNDE4WiIsImlzcyI6InJ1ZGRlciIsImp0aSI6IjZkMjNkNjFjLWQwZDQtNDkyYy1iYmU3LTQzODYwN2Q1ODEwYiIsIm5iZiI6IjIwMjItMDUtMDlUMDU6MjE6NTYuNDQwNDgwNDE4WiIsInN1YiI6ImFkbWluIn0.MjKpJ4lQzbjfzeRiTPZki72zvqRpiZOFzOBt14PvMDo

# 添加仓库地址
$ tkeel repo add tkeel https://tkeel-io.github.io/helm-charts
$ tkeel repo add lunz https://lunz1207.github.io/helm-charts/

# 安装插件
$ echo "console-portal-admin" |  xargs -I {} tkeel plugin install repo-name/{}@0.5.0 {}
$ echo "console-portal-tenant" |  xargs -I {} tkeel plugin install repo-name/{}@0.5.0 {}
$ echo "console-portal-admin" |  xargs -I {} tkeel plugin install repo-name/{}@0.5.0 {}
```
- other
- 

#### 7 kubesphere
##### 7.1 install
##### 7.2 access
```
#Console: http://172.18.0.2:30880
#Account: admin
#Password: P@88w0rd
#NewPassWord: Qwer9876

$ kubectl get pods -n kubesphere-system
$ kubectl get svc -n kubesphere-system
$ kubectl port-forward svc/ks-console 8888:80  -n kubesphere-system 
```
#### 8 tkeel
```
# addons_identity
{
    "res":
    {
        "ret": 0,
        "msg": "ok"
    },
    "plugin_id": "tkeel-device-template",
    "version": "v0.0.1",
    "tkeel_version": "v0.4.0",
    "implemented_plugin":
    [
        {
            "plugin":
            {
                "id": "tkeel-device",
                "version": "v0.4.1"
            },
            "addons":
            [
                {
                    "addons_point": "device-schema-change",
                    "implemented_endpoint": "ping"
                }
            ]
        }
    ]
}
```
```
# plugin_route
{
    "":
    {
        "version": "1"
    },
    "console-plugin-admin-custom-config":
    {
        "id": "console-plugin-admin-custom-config",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-plugin-admin-notification-configs":
    {
        "id": "console-plugin-admin-notification-configs",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-plugin-admin-plugins":
    {
        "id": "console-plugin-admin-plugins",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-plugin-admin-service-monitoring":
    {
        "id": "console-plugin-admin-service-monitoring",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-plugin-admin-tenants":
    {
        "id": "console-plugin-admin-tenants",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-plugin-admin-usage-statistics":
    {
        "id": "console-plugin-admin-usage-statistics",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-plugin-tenant-alarm-policy":
    {
        "id": "console-plugin-tenant-alarm-policy",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-plugin-tenant-alarms":
    {
        "id": "console-plugin-tenant-alarms",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-plugin-tenant-data-query":
    {
        "id": "console-plugin-tenant-data-query",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-plugin-tenant-data-subscription":
    {
        "id": "console-plugin-tenant-data-subscription",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-plugin-tenant-device-templates":
    {
        "id": "console-plugin-tenant-device-templates",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-plugin-tenant-devices":
    {
        "id": "console-plugin-tenant-devices",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-plugin-tenant-networks":
    {
        "id": "console-plugin-tenant-networks",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-plugin-tenant-notification-objects":
    {
        "id": "console-plugin-tenant-notification-objects",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-plugin-tenant-plugins":
    {
        "id": "console-plugin-tenant-plugins",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-plugin-tenant-roles":
    {
        "id": "console-plugin-tenant-roles",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-plugin-tenant-routing-rules":
    {
        "id": "console-plugin-tenant-routing-rules",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-plugin-tenant-users":
    {
        "id": "console-plugin-tenant-users",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-portal-admin":
    {
        "id": "console-portal-admin",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "console-portal-tenant":
    {
        "id": "console-portal-tenant",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "core-broker":
    {
        "id": "core-broker",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "fluxswitch":
    {
        "id": "fluxswitch",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "iothub":
    {
        "id": "iothub",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "rule-manager":
    {
        "id": "rule-manager",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    },
    "tkeel-alarm":
    {
        "id": "tkeel-alarm",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "implemented_plugin":
        [
            "tkeel-device"
        ],
        "version": "1"
    },
    "tkeel-device":
    {
        "id": "tkeel-device",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1",
        "register_addons":
        {
            "device-schema-change": "tkeel-alarm/v1/notify/update"
        }
    },
    "tkeel-monitor":
    {
        "id": "tkeel-monitor",
        "status": 3,
        "tkeel_version": "v0.4.0",
        "version": "1"
    }
}
```