# 一、简介

Kubernetes 容器编排已越来越被大家关注，然而使用 Kubernetes 的门槛却依然很高，主要体现在这几个方面：

- 集群的安装复杂，出错概率大
- Kubernetes相较于容器化，引入了许多新的概念，学习难度高
- 需要手工编写 YAML 文件，难以在多环境下管理
- 缺少好的实战案例可以参考

Kuboard，是一款免费的 Kubernetes 图形化管理工具，Kuboard 力图帮助用户快速在 Kubernetes 上落地微服务。

#### [官方配置](https://kuboard.cn/install-script/kuboard.yaml)

```bash
mkdir kuboard && cd kuboard
```

#### 配置清单

RBAC

```yaml
cat << 'EOF' >rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kuboard-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kuboard-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kuboard-user
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kuboard-viewer
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kuboard-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
- kind: ServiceAccount
  name: kuboard-viewer
  namespace: kube-system
EOF
```

Deployment

```yaml
cat << 'EOF' >dp.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kuboard
  namespace: kube-system
  labels:
    k8s.kuboard.cn/name: kuboard
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s.kuboard.cn/name: kuboard
  template:
    metadata:
      labels:
        k8s.kuboard.cn/name: kuboard
    spec:
      containers:
      - name: kuboard
        image: eipwork/kuboard:latest
        imagePullPolicy: Always
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
EOF
```

Service

```yaml
cat << 'EOF' >svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: kuboard
  namespace: kube-system
spec:
  ports:
  - name: http
    port: 80
    targetPort: 80
  selector:
    k8s.kuboard.cn/name: kuboard
EOF
```

Ingress

```yaml
cat << EOF >ingress.yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: kuboard
  namespace: kube-system
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`kuboard.wzxmt.com`) && PathPrefix(`/`)
    kind: Rule
    services:
    - name: kuboard
      port: 80
EOF
```

#### 应用资源清单

```bash
kubectl apply -f ./
```

#### 查看 Kuboard 运行状态：

```bash
kubectl get pods -n kube-system -l k8s.kuboard.cn/name=kuboard
```

#### 输出结果如下所示：

```bash
NAME                       READY   STATUS    RESTARTS   AGE
kuboard-7bb89b4cc4-s9sns   1/1     Running   0          3h39
```

#### [获取Token](此Token拥有 ClusterAdmin 的权限，可以执行所有操作)

```bash
echo $(kubectl -n kube-system get secret $(kubectl -n kube-system get secret | grep kuboard-user | awk '{print $1}') -o go-template='{{.data.token}}' | base64 -d)
```

#### 取输出信息中 token 字段

```bash
eyJhbGciOiJSUzI1NiIsImtpZCI6IjRoSTJMX2hJZF9DTnZ6N1RtYlY3ZEplZmdwZXlkQk52RWNrZS01M0RpWGMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJvYXJkLXVzZXItdG9rZW4tbXo0Z3ciLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoia3Vib2FyZC11c2VyIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiNmExNzUzZjAtMDY4NS00MTJmLThkYTgtOGNlNjEwMjRiNWYzIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmt1Ym9hcmQtdXNlciJ9.GfyJN2jnTcXvgS5yvjxc7Y6slfBAqiNBdQipJhDIqha8TI8AZU22aMhmkyzAWNIoEwNXuitALRK__1EVqdKKR3Qe7sAKCAWAHTYoRQ0xb4bRzPHlCcy5BTC9Y69TjuKJsF1O_Bc2N4HvPmg25C8G3MSx-1ECMPuev8UlOKzqLu6sNSCFOG_96-u7S0SMSiuicqGPhwlM_WngPWio3WWWetC2zLHsh59gKPxBwZpLG_-2PZURqw81qr_Pb5G4Ve9f4nc-C3Wc_TYxK2R-8dYNfMDRX8dJ8mkUvTshnwF4yGbFHTYlNR-e_2Xv4dXBEICiqBIyuIba-yLO5H2N_GkWew
```

#### dns添加域名解析

```bash
kuboard	60 IN A 10.0.0.50
```

#### 访问http://kuboard.wzxmt.com