[toc]
# PV、PVC概述
管理存储是管理计算的一个明显问题。PersistentVolume子系统为用户和管理员提供了一个API，用于抽象如何根据消费方式提供存储的详细信息。于是引入了两个新的API资源：PersistentVolume和PersistentVolumeClaim
>PersistentVolume（PV）是集群中已由管理员配置的一段网络存储。 集群中的资源就像一个节点是一个集群资源。 PV是诸如卷之类的卷插件，但是具有独立于使用PV的任何单个pod的生命周期。 该API对象包含存储的实现细节，即NFS，iSCSI或云提供商特定的存储系统。 

>PersistentVolumeClaim（PVC）是用户存储的请求。 它类似于pod。Pod消耗节点资源，PVC消耗存储资源。 pod可以请求特定级别的资源（CPU和内存）。 权限要求可以请求特定的大小和访问模式。

>虽然PersistentVolumeClaims允许用户使用抽象存储资源，但是常见的是，用户需要具有不同属性（如性能）的PersistentVolumes，用于不同的问题。 管理员需要能够提供多种不同于PersistentVolumes，而不仅仅是大小和访问模式，而不会使用户了解这些卷的实现细节。 对于这些需求，存在StorageClass资源。

>StorageClass为集群提供了一种描述他们提供的存储的“类”的方法。 不同的类可能映射到服务质量级别，或备份策略，或者由群集管理员确定的任意策略。 Kubernetes本身对于什么类别代表是不言而喻的。 这个概念有时在其他存储系统中称为“配置文件”
# POD动态供给
>动态供给主要是能够自动帮你创建pv，需要多大的空间就创建多大的pv。k8s帮助创建pv，创建pvc就直接api调用存储类来寻找pv。

>如果是存储静态供给的话，会需要我们手动去创建pv，如果没有足够的资源，找不到合适的pv，那么pod就会处于pending等待的状态。而动态供给主要的一个实现就是StorageClass存储对象，其实它就是声明你使用哪个存储，然后帮你去连接，再帮你去自动创建pv。
# POD使用RBD做为持久数据卷
### 安装与配置
RBD支持ReadWriteOnce，ReadOnlyMany两种模式
1、配置rbd-provisioner
```
cat << EOF>external-storage-rbd-provisioner.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rbd-provisioner
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-provisioner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["kube-dns"]
    verbs: ["list", "get"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rbd-provisioner
subjects:
  - kind: ServiceAccount
    name: rbd-provisioner
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: rbd-provisioner
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: rbd-provisioner
  namespace: kube-system
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rbd-provisioner
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: rbd-provisioner
subjects:
- kind: ServiceAccount
  name: rbd-provisioner
  namespace: kube-system

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rbd-provisioner
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: rbd-provisioner
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: rbd-provisioner
    spec:
      containers:
      - name: rbd-provisioner
        image: "quay.io/external_storage/rbd-provisioner:v2.0.0-k8s1.11"
        env:
        - name: PROVISIONER_NAME
          value: ceph.com/rbd
      serviceAccount: rbd-provisioner
EOF
kubectl apply -f external-storage-rbd-provisioner.yaml
```
2、配置storageclass

```
1、创建pod时，kubelet需要使用rbd命令去检测和挂载pv对应的ceph image，所以要在所有的worker节点安装ceph客户端ceph-common。
yum -y install ceph-common
将ceph的ceph.client.admin.keyring和ceph.conf文件拷贝到worker节点的/etc/ceph目录下
2、创建 osd pool 在ceph的mon或者admin节点
ceph osd pool create kube 128 128 
ceph osd pool ls
3、创建k8s访问ceph的用户 在ceph的mon或者admin节点
ceph auth get-or-create client.kube mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=kube' -o ceph.client.kube.keyring
4、查看key 在ceph的mon或者admin节点
ceph auth get-key client.admin
ceph auth get-key client.kube
5、创建 admin secret
kubectl create secret generic ceph-secret \
--type="kubernetes.io/rbd" \
--from-literal=key=AQA3SVRf7OdUBRAATiGOfVJ/bCbx3X0RG9oR8A== \
--namespace=kube-system
6、在 default 命名空间创建pvc用于访问ceph的 secret
kubectl create secret generic ceph-user-secret \
--type="kubernetes.io/rbd" \
--from-literal=key=AQCDyFRf5lCfMxAAx4pDefsQ1kCDP/4BRKA3eQ== \
--namespace=default
```
3、配置StorageClass
```
cat<< 'EOF' >storageclass-ceph-rdb.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: dynamic-ceph-rdb
provisioner: ceph.com/rbd
parameters:
  monitors: 10.0.0.61:6789,10.0.0.62:6789,10.0.0.63:6789
  adminId: admin
  adminSecretName: ceph-secret
  adminSecretNamespace: kube-system
  pool: kube
  userId: kube
  userSecretName: ceph-user-secret
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"
EOF
```
4、创建yaml
```
kubectl apply -f storageclass-ceph-rdb.yaml
```
5、查看sc
```
kubectl get storageclasses 
```
### 测试使用
1、创建pvc测试
```
cat<< 'EOF' >ceph-rdb-pvc-test.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ceph-rdb-claim
spec:
  accessModes:     
    - ReadWriteOnce
  storageClassName: dynamic-ceph-rdb
  resources:
    requests:
      storage: 2Gi
EOF
kubectl apply -f ceph-rdb-pvc-test.yaml
```
2、查看
```
kubectl get pvc
kubectl get pv
```
3、创建 nginx pod 挂载测试
```
cat<< 'EOF' >nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod1
  labels:
    name: nginx-pod1
spec:
  containers:
  - name: nginx-pod1
    image: nginx:alpine
    ports:
    - name: web
      containerPort: 80
    volumeMounts:
    - name: ceph-rdb
      mountPath: /usr/share/nginx/html
  volumes:
  - name: ceph-rdb
    persistentVolumeClaim:
      claimName: ceph-rdb-claim
EOF
kubectl apply -f nginx-pod.yaml
```
4、查看
```
kubectl get pods -o wide
```
5、修改文件内容
```
kubectl exec -ti nginx-pod1 -- /bin/sh -c 'echo this is from Ceph RBD!!! > /usr/share/nginx/html/index.html'
```
6、访问测试
```
curl http://$podip
```
7、清理
```
kubectl delete -f nginx-pod.yaml
kubectl delete -f ceph-rdb-pvc-test.yaml
```
# POD使用CephFS做为持久数据卷

CephFS方式支持k8s的pv的3种访问模式ReadWriteOnce，ReadOnlyMany ，ReadWriteMany
## Ceph端创建CephFS pool

1、如下操作在ceph的mon或者admin节点
CephFS需要使用两个Pool来分别存储数据和元数据
```
ceph osd pool create fs_data 128
ceph osd pool create fs_metadata 128
ceph osd lspools
```
2、创建一个CephFS
```
ceph fs new cephfs fs_metadata fs_data
```
3、查看
```
ceph fs ls
```

## 部署 cephfs-provisioner

1、使用社区提供的cephfs-provisioner
```
cat<< 'EOF'  >external-storage-cephfs-provisioner.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cephfs-provisioner
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cephfs-provisioner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["create", "get", "delete"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cephfs-provisioner
subjects:
  - kind: ServiceAccount
    name: cephfs-provisioner
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cephfs-provisioner
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cephfs-provisioner
  namespace: kube-system
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["create", "get", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cephfs-provisioner
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cephfs-provisioner
subjects:
- kind: ServiceAccount
  name: cephfs-provisioner
  namespace: kube-system

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cephfs-provisioner
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: cephfs-provisioner
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: cephfs-provisioner
    spec:
      containers:
      - name: cephfs-provisioner
        image: "quay.io/external_storage/cephfs-provisioner:latest"
        env:
        - name: PROVISIONER_NAME
          value: ceph.com/cephfs
        command:
        - "/usr/local/bin/cephfs-provisioner"
        args:
        - "-id=cephfs-provisioner-1"
      serviceAccount: cephfs-provisioner
EOF
kubectl apply -f external-storage-cephfs-provisioner.yaml
```
2、查看状态 等待running之后 再进行后续的操作
```
kubectl get pod -n kube-system
```
## 配置 storageclass
1、查看key 在ceph的mon或者admin节点
```
ceph auth get-key client.admin
```
2、创建 admin secret
```
kubectl create secret generic admin-ceph-secret \
--type="kubernetes.io/rbd" \
--from-literal=key=AQA3SVRf7OdUBRAATiGOfVJ/bCbx3X0RG9oR8A== \
--namespace=kube-system
```
3、查看 secret

```
kubectl get secret ceph-secret -n kube-system -o yaml
```
4、配置 StorageClass

```
cat >storageclass-cephfs.yaml<<EOF
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: dynamic-cephfs
provisioner: ceph.com/cephfs
parameters:
    monitors: 10.0.0.61:6789,10.0.0.62:6789,10.0.0.63:6789
    adminId: admin
    adminSecretName: admin-ceph-secret
    adminSecretNamespace: "kube-system"
    claimRoot: /volumes/kubernetes
EOF
```
5、创建

```
kubectl apply -f storageclass-cephfs.yaml
```
6、查看
```
kubectl get sc
```
## 测试使用

1、创建pvc测试
```
cat >cephfs-pvc-test.yaml<<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: cephfs-claim
spec:
  accessModes:     
    - ReadWriteMany
  storageClassName: dynamic-cephfs
  resources:
    requests:
      storage: 2Gi
EOF
kubectl apply -f cephfs-pvc-test.yaml
```
2、查看
```
kubectl get pvc
kubectl get pv
```
3、创建 nginx pod 挂载测试
```
cat >nginx-pod.yaml<<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod2
  labels:
    name: nginx-pod2
spec:
  containers:
  - name: nginx-pod2
    image: nginx
    ports:
    - name: web
      containerPort: 80
    volumeMounts:
    - name: cephfs
      mountPath: /usr/share/nginx/html
  volumes:
  - name: cephfs
    persistentVolumeClaim:
      claimName: cephfs-claim
EOF
kubectl apply -f nginx-pod.yaml
```
 4、查看
 ```
kubectl get pods -o wide
 ```
 5、修改文件内容
 ```
kubectl exec -ti nginx-pod2 -- /bin/sh -c 'echo This is from CephFS!!! > /usr/share/nginx/html/index.html'
 ```
6、访问pod测试
```
curl http://$podip
```
7、清理
```
kubectl delete -f nginx-pod.yaml
kubectl delete -f cephfs-pvc-test.yaml
```