---
title: "kubernetes-Pod-Security-Policy"
draft: false
toc: true
description: "kubernetes安全准入控制"
date: 2023-06-11
featured_image: /images/kubernetes.png
categories: []
tags: [kubernetes, k8s, psp, security]
---
K8s PSP（Pod Security Policy）是 Kubernetes 中的一个安全特性，用于限制 Pod 的权限和访问能力，以确保 Pod 在运行时不会破坏 Kubernetes 集群的安全。
PSP 为管理员提供了一种机制，可以在 Kubernetes 集群中定义强制执行的安全策略。这些安全策略可以限制容器的能力，例如禁止使用特定的 Linux 系统调用、禁止从容器内部将网络流量转发到集群内部等等。管理员可以创建多个 PSP 并将其分配给不同的用户或命名空间，以便根据需要为不同的应用程序或用户组设置不同的安全策略。
要启用 Pod 安全策略，必须确保 Kubernetes 集群已启用 RBAC（基于角色的访问控制）。此外，还需要为 Pod 安全策略定义 ClusterRole 和 ClusterRoleBinding，以授予管理员创建、更新和删除 Pod 安全策略的权限。<!--more-->
#### 1 启用PSP的步骤
启用 PodSecurityPolicy 特性门户：Kubernetes v1.21 及更高版本默认情况下已启用 PodSecurityPolicy 特性门户。如果您的 Kubernetes 版本低于 v1.21，则需要手动启用此特性。

1. 创建 PodSecurityPolicy 对象：可以使用 YAML 或 JSON 文件定义 PodSecurityPolicy 的规则。该文件应包含 PodSecurityPolicy 规则的详细信息，例如容器的安全上下文、主机网络和文件系统访问权限等内容。

2. 创建 ClusterRole 和 ClusterRoleBinding 对象：创建 ClusterRole 并授予适当的权限来管理 PodSecurityPolicy。然后，为将此角色分配给用户或服务帐户创建 ClusterRoleBinding。

3. 将 PodSecurityPolicy 分配给命名空间：PodSecurityPolicy 可以通过名称引用，然后将其分配给命名空间。这将确保所有在该命名空间下创建的 Pod 都遵循 PodSecurityPolicy 中定义的规则。

4. 部署 Pod：一旦 PodSecurityPolicy 被分配到了相应的命名空间中，就可以使用它们的规则来部署 Pod。将所需的安全上下文和其他配置添加到 Pod 定义中，以确保它们符合 PodSecurityPolicy 中定义的规则。

#### 2 配置PSP最佳实践

##### 2.1 创建restricted和privileged两种PSP规则
```shell
cat <<EOF | kubectl apply -f -
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  seLinux:
    rule: RunAsAny
  runAsUser:
    rule: MustRunAsNonRoot
  fsGroup:
    rule: MustRunAs
    ranges:
    - min: 1
      max: 65535
  supplementalGroups:
    rule: MustRunAs
    ranges:
    - min: 1
      max: 65535
  readOnlyRootFilesystem: true
  hostNetwork: false
  hostIPC: false
  hostPID: false
  volumes:       
  - 'configMap'
  - 'emptyDir'
  - 'projected'
  - 'secret'
  - 'downwardAPI'
  - 'csi'
  - 'persistentVolumeClaim'
EOF
```
```shell
cat <<EOF| kubectl apply -f -
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: privileged-psp
spec:
  privileged: true
  hostNetwork: true
  allowPrivilegeEscalation: true
  defaultAllowPrivilegeEscalation: true
  hostPID: true
  hostIPC: true
  runAsUser:
    rule: RunAsAny
  fsGroup:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  volumes:
  - '*'
  allowedCapabilities:
  - '*'  
EOF
```
##### 2.2 创建RBAC权限
```shell
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: restricted-psp
rules:
- apiGroups:
  - policy
  resources:
  - podsecuritypolicies
  verbs:
  - use
  resourceNames:
  - restricted-psp
EOF
```
```shell
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: privileged-psp
rules:
- apiGroups:
  - policy
  resources:
  - podsecuritypolicies
  verbs:
  - use
  resourceNames:
  - privileged-psp
EOF
```
##### 2.3 将ClusterRole绑定到sa/group/namespace
###### 2.3.1 将restricted-psp绑定到所有认证用户
```shell
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: psp-global
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: restricted-psp
subjects:
- kind: Group
  name: system:authenticated
  apiGroup: rbac.authorization.k8s.io
EOF
```
###### 2.3.2 将privileged-psp绑定到某个namespace
```shell
cat <<EOF | kubectl apply -f -
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
name: psp-kubespere-system
subjects:
- kind: Group
  name: system:serviceaccounts:kubesphere-system    
  apiGroup: rbac.authorization.k8s.io
  roleRef:
  kind: ClusterRole
  name: privileged-psp
  apiGroup: rbac.authorization.k8s.io
  EOF
```
###### 2.3.3 启用admission插件
```text
# /etc/kubernetes/manifests/kube-apiserver.yaml
# 在 enable-admission-plugins 中 添加 PodSecurityPolicy
# enable-admission-plugins=PodSecurityPolicy
```

###### 2.4 验证 PSP 是否生效
```shell
cat <<EOF| kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
spec:
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      securityContext:
        runAsUser: 0
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
EOF
```
```shell
# 查看deploy状态
# kubectl get deploy nginx -o=jsonpath={.status}
```
```shell
cat <<EOF| kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: kubesphere-system
spec:
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      securityContext:
        runAsUser: 0
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
EOF
```
```shell
# 查看deploy状态
# kubectl -n kubesphere-system get deploy nginx -o=jsonpath={.status}
```
###### 2.5 验证sa可以访问PSP
1. 创建namespace
    ```shell
    kubectl create namespace restricted-ns
    ```
2. 创建sa
    ```shell
    kubectl create sa restricted-sa -n restricted-ns
    ```
3. 检查sa是否有psp的访问权限
    ```shell
    kubectl auth can-i use podsecuritypolicy/restricted-psp \
    --as=system:serviceaccount:restricted-ns:restricted-sa -n restricted-ns
    ```
