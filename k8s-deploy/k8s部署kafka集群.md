# 一、简介



## 1.1、statefulset控制器简介

statefulset控制器是有状态应用副本集；在k8s集群statefulset用于管理以下副本集：

- 稳定且唯一的网络标识符
- 稳定且持久的存储
- 有序，平滑的部署和扩展
- 有序，平滑的删除和终止
- 有序的滚动更新statefulset包含三个组件：headless service、StatefulSet，volumeClaimTemplate
- headless service：确保解析名称直达后端pod
- volumeClaimTemplate：卷申请模板，每创建一个pod时，自动申请一个pvc，从而请求绑定pv
- StatefulSet：控制器



# 二、部署

## 2.0、zk集群部署

### 2.1、创建动态pv

### 2.1.1、准备

```bash
mkdir zk && cd zk
kubectl create ns infra
```

### 2.1.2、定义资源

```yaml
cat<< 'EOF' >zk-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: zk-data
  namespace: infra
provisioner: fuseim.pri/ifs
EOF
```

### 2.1.3、创建StorageClass

```
kubectl  apply  -f  zk-sc.yaml
```

### 2.1.4、查看StorageClass

```bash
[root@manage zk]# kubectl get StorageClass 
NAME             PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
zk-data          fuseim.pri/ifs   Delete          Immediate           false                  23m
```

### 2.2.1、官方资源清单修改

官方资源清单：https://kubernetes.io/zh/docs/tutorials/stateful-application/zookeeper/

StatefulSet

```yaml
cat<< 'EOF' >StatefulSet.yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
  namespace: infra
spec:
  selector:
    matchLabels:
      app: zk
  maxUnavailable: 1
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zk
  namespace: infra
spec:
  selector:
    matchLabels:
      app: zk
  serviceName: zk-hs
  replicas: 3
  updateStrategy:
    type: RollingUpdate
  podManagementPolicy: OrderedReady
  template:
    metadata:
      labels:
        app: zk
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                    - zk
              topologyKey: "kubernetes.io/hostname"
      containers:
      - name: kubernetes-zookeeper
        imagePullPolicy: IfNotPresent
        image: "mirrorgcrio/kubernetes-zookeeper:1.0-3.4.10"
        resources:
          requests:
            memory: "500Mi"
            cpu: "0.5"
        ports:
        - containerPort: 2181
          name: client
        - containerPort: 2888
          name: server
        - containerPort: 3888
          name: leader-election
        command:
        - sh
        - -c
        - "start-zookeeper \
          --servers=3 \
          --data_dir=/var/lib/zookeeper/data \
          --data_log_dir=/var/lib/zookeeper/data/log \
          --conf_dir=/opt/zookeeper/conf \
          --client_port=2181 \
          --election_port=3888 \
          --server_port=2888 \
          --tick_time=2000 \
          --init_limit=10 \
          --sync_limit=5 \
          --heap=512M \
          --max_client_cnxns=60 \
          --snap_retain_count=3 \
          --purge_interval=12 \
          --max_session_timeout=40000 \
          --min_session_timeout=4000 \
          --log_level=INFO"
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - "zookeeper-ready 2181"
          initialDelaySeconds: 10
          timeoutSeconds: 5
        livenessProbe:
          exec:
            command:
            - sh
            - -c
            - "zookeeper-ready 2181"
          initialDelaySeconds: 10
          timeoutSeconds: 5
        volumeMounts:
        - name: zk-data-storage
          mountPath: /var/lib/zookeeper
  volumeClaimTemplates:
  - metadata:
      name: zk-data-storage
    spec:
      accessModes: 
      - ReadWriteOnce
      storageClassName: zk-data
      resources:
        requests:
          storage: 10Gi
EOF
```

Service

```yaml
cat<< 'EOF' >svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: zk-hs
  namespace: infra
  labels:
    app: zk
spec:
  ports:
  - port: 2888
    name: server
  - port: 3888
    name: leader-election
  clusterIP: None
  selector:
    app: zk
---
apiVersion: v1
kind: Service
metadata:
  name: zk-cs
  namespace: infra
  labels:
    app: zk
spec:
  ports:
  - port: 2181
    name: client
  selector:
    app: zk
EOF
```

### 2.2.2、创建zk集群

```bash
kubectl  apply  -f  ./
```

### 2.2.3、查看pod

```bash
[root@manage ~]# kubectl get pod -n infra 
NAME                             READY   STATUS    RESTARTS   AGE
zk-0                             1/1     Running   0          49m
zk-1                             1/1     Running   0          50m
zk-2                             1/1     Running   0          51m
```

### 2.2.4、查看pvc

```bash
[root@manage ~]# kubectl get pvc -n infra
NAME                  STATUS   VOLUME                               CAPACITY ACCESS MODES   STORAGECLASS   AGE
zk-data-storage-zk-0  Bound pvc-1bb9ea4c-a3e6-4f7a-b281-b5c447309c71  10Gi    RWO   zk-data        106m
zk-data-storage-zk-1  Bound pvc-d6140259-3141-44ce-ada6-ee95b974949e  10Gi    RWO   zk-data        102m
zk-data-storage-zk-2  Bound pvc-c6cdf100-3397-45e4-ae37-23402274d416  10Gi    RWO   zk-data        101m
```

### 2.2.5、查看zk配置

```bash
[root@manage zk]# kubectl exec -n infra zk-0 -- cat /opt/zookeeper/conf/zoo.cfg
#This file was autogenerated DO NOT EDIT
clientPort=2181
dataDir=/var/lib/zookeeper/data
dataLogDir=/var/lib/zookeeper/data/log
tickTime=2000
initLimit=10
syncLimit=5
maxClientCnxns=60
minSessionTimeout=4000
maxSessionTimeout=40000
autopurge.snapRetainCount=3
autopurge.purgeInteval=12
server.1=zk-0.zk-hs.infra.svc.cluster.local:2888:3888
server.2=zk-1.zk-hs.infra.svc.cluster.local:2888:3888
server.3=zk-2.zk-hs.infra.svc.cluster.local:2888:3888
```

### 2.2.6、查看集群状态

```bash
[root@manage ~]# kubectl exec -n infra zk-0 zkServer.sh status
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.
ZooKeeper JMX enabled by default
Using config: /usr/bin/../etc/zookeeper/zoo.cfg
Mode: follower
[root@manage ~]# kubectl exec -n infra zk-1 zkServer.sh status
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.
ZooKeeper JMX enabled by default
Using config: /usr/bin/../etc/zookeeper/zoo.cfg
Mode: leader
[root@manage ~]# kubectl exec -n infra zk-2 zkServer.sh status
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.
ZooKeeper JMX enabled by default
Using config: /usr/bin/../etc/zookeeper/zoo.cfg
Mode: follower
```

### 2.2.7 、扩缩容

修改配置

```
kubectl -n infra edit statefulsets zk
```

--servers=3与replicas: 3保持一致

## 3.0、kafka集群部署

```bash
mkdir kafka && cd kafka
```

### 3.1.1、定义StorageClass资源

```yaml
cat<< 'EOF' > kafka-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: kafka-data
  namespace: infra
provisioner: fuseim.pri/ifs
EOF
```

### 3.1.2、创建StorageClass

```bash
kubectl  apply  -f  kafka-sc.yaml
```

### 3.1.3、查看StorageClass

```bash
[root@manage kafka]# kubectl get StorageClass 
NAME             PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
kafka-data          fuseim.pri/ifs   Delete          Immediate           false                  23m
```

### 3.2.1、资源清单

StatefulSet

```yaml
cat<< 'EOF' > kafka-sfs.yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: kafka-pdb
  namespace: infra
spec:
  selector:
    matchLabels:
      app: kafka
  maxUnavailable: 1
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: infra
spec:
  serviceName: kafka-hs
  replicas: 3
  podManagementPolicy: Parallel
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      terminationGracePeriodSeconds: 300
      containers:
      - name: k8skafka
        imagePullPolicy: IfNotPresent
        image: mirrorgooglecontainers/kubernetes-kafka:1.0-10.2.1
        resources:
          requests:
            memory: "256Mi"
            cpu: "0.1"
        ports:
        - containerPort: 9093
          name: server
        command:
        - sh
        - -c
        - "exec kafka-server-start.sh /opt/kafka/config/server.properties --override broker.id=${HOSTNAME##*-} \
          --override listeners=PLAINTEXT://:9093 \
          --override zookeeper.connect=zk-0.zk-hs:2181,zk-1.zk-hs:2181,zk-2.zk-hs:2181 \
          --override log.dir=/var/lib/kafka \
          --override auto.create.topics.enable=true \
          --override auto.leader.rebalance.enable=true \
          --override background.threads=10 \
          --override compression.type=producer \
          --override delete.topic.enable=true \
          --override leader.imbalance.check.interval.seconds=300 \
          --override leader.imbalance.per.broker.percentage=10 \
          --override log.flush.interval.messages=9223372036854775807 \
          --override log.flush.offset.checkpoint.interval.ms=60000 \
          --override log.flush.scheduler.interval.ms=9223372036854775807 \
          --override log.retention.bytes=-1 \
          --override log.retention.hours=168 \
          --override log.roll.hours=168 \
          --override log.roll.jitter.hours=0 \
          --override log.segment.bytes=1073741824 \
          --override log.segment.delete.delay.ms=60000 \
          --override message.max.bytes=1000012 \
          --override min.insync.replicas=1 \
          --override num.io.threads=8 \
          --override num.network.threads=3 \
          --override num.recovery.threads.per.data.dir=1 \
          --override num.replica.fetchers=1 \
          --override offset.metadata.max.bytes=4096 \
          --override offsets.commit.required.acks=-1 \
          --override offsets.commit.timeout.ms=5000 \
          --override offsets.load.buffer.size=5242880 \
          --override offsets.retention.check.interval.ms=600000 \
          --override offsets.retention.minutes=1440 \
          --override offsets.topic.compression.codec=0 \
          --override offsets.topic.num.partitions=50 \
          --override offsets.topic.replication.factor=3 \
          --override offsets.topic.segment.bytes=104857600 \
          --override queued.max.requests=500 \
          --override quota.consumer.default=9223372036854775807 \
          --override quota.producer.default=9223372036854775807 \
          --override replica.fetch.min.bytes=1 \
          --override replica.fetch.wait.max.ms=500 \
          --override replica.high.watermark.checkpoint.interval.ms=5000 \
          --override replica.lag.time.max.ms=10000 \
          --override replica.socket.receive.buffer.bytes=65536 \
          --override replica.socket.timeout.ms=30000 \
          --override request.timeout.ms=30000 \
          --override socket.receive.buffer.bytes=102400 \
          --override socket.request.max.bytes=104857600 \
          --override socket.send.buffer.bytes=102400 \
          --override unclean.leader.election.enable=true \
          --override zookeeper.session.timeout.ms=6000 \
          --override zookeeper.set.acl=false \
          --override broker.id.generation.enable=true \
          --override connections.max.idle.ms=600000 \
          --override controlled.shutdown.enable=true \
          --override controlled.shutdown.max.retries=3 \
          --override controlled.shutdown.retry.backoff.ms=5000 \
          --override controller.socket.timeout.ms=30000 \
          --override default.replication.factor=1 \
          --override fetch.purgatory.purge.interval.requests=1000 \
          --override group.max.session.timeout.ms=300000 \
          --override group.min.session.timeout.ms=6000 \
          --override inter.broker.protocol.version=0.10.2-IV0 \
          --override log.cleaner.backoff.ms=15000 \
          --override log.cleaner.dedupe.buffer.size=134217728 \
          --override log.cleaner.delete.retention.ms=86400000 \
          --override log.cleaner.enable=true \
          --override log.cleaner.io.buffer.load.factor=0.9 \
          --override log.cleaner.io.buffer.size=524288 \
          --override log.cleaner.io.max.bytes.per.second=1.7976931348623157E308 \
          --override log.cleaner.min.cleanable.ratio=0.5 \
          --override log.cleaner.min.compaction.lag.ms=0 \
          --override log.cleaner.threads=1 \
          --override log.cleanup.policy=delete \
          --override log.index.interval.bytes=4096 \
          --override log.index.size.max.bytes=10485760 \
          --override log.message.timestamp.difference.max.ms=9223372036854775807 \
          --override log.message.timestamp.type=CreateTime \
          --override log.preallocate=false \
          --override log.retention.check.interval.ms=300000 \
          --override max.connections.per.ip=2147483647 \
          --override num.partitions=3 \
          --override producer.purgatory.purge.interval.requests=1000 \
          --override replica.fetch.backoff.ms=1000 \
          --override replica.fetch.max.bytes=1048576 \
          --override replica.fetch.response.max.bytes=10485760 \
          --override reserved.broker.max.id=1000 "
        env:
        - name: KAFKA_HEAP_OPTS
          value : "-Xmx256M -Xms256M"
        - name: KAFKA_OPTS
          value: "-Dlogging.level=INFO"
        volumeMounts:
        - name: kafka-data-storage
          mountPath: /var/lib/kafka
        readinessProbe:
          exec:
           command:
            - sh
            - -c
            - "/opt/kafka/bin/kafka-broker-api-versions.sh --bootstrap-server=localhost:9093"
      securityContext:
        runAsUser: 1000
        fsGroup: 1000
  volumeClaimTemplates:
  - metadata:
      name: kafka-data-storage
    spec:
      accessModes: 
      - ReadWriteOnce
      storageClassName: kafka-data
      resources:
        requests:
          storage: 10Gi
EOF
```

Service

```
cat<< 'EOF' > kafka-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-hs
  namespace: infra
  labels:
    app: kafka
spec:
  ports:
  - port: 9093
    name: server
  clusterIP: None
  selector:
    app: kafka
EOF
```

### 3.2.2、创建zk集群

```bash
kubectl  apply  -f  ./
```

### 3.2.3、查看pod

```bash
[root@manage ~]# kubectl get pod -n infra 
NAME                             READY   STATUS    RESTARTS   AGE
kafka-0                          1/1     Running   0          18m
kafka-1                          1/1     Running   0          18m
kafka-2                          1/1     Running   0          18m
```

### 3.2.4、查看pvc

```bash
[root@manage ~]# kubectl get pvc -n infra
NAME                      STATUS VOLUME                               CAPACITY ACCESS MODES   STORAGECLASS   AGE
kafka-data-storage-kafka-0 Bound pvc-915fc0a7-62a5-45cb-bfa6-b4f81668964e 10Gi   RWO  kafka-data     29m
kafka-data-storage-kafka-1 Bound pvc-51d25e54-5253-4c11-b89e-dc030e9f6046 10Gi   RWO  kafka-data     29m
kafka-data-storage-kafka-2 Bound pvc-243860cf-ea7c-4354-b099-82bda11a718e 10Gi   RWO  kafka-data     29m
```

### 3.2.5、通过zookeeper查看broker

```bash
[root@manage ~]# kubectl exec -ti zk-1 -n infra zkCli.sh
[zk: localhost:2181(CONNECTED) 1] get /brokers/ids/0
{"listener_security_protocol_map":{"PLAINTEXT":"PLAINTEXT"},"endpoints":["PLAINTEXT://kafka-0.kafka-hs.infra.svc.cluster.local:9093"],"jmx_port":-1,"host":"kafka-0.kafka-hs.infra.svc.cluster.local","timestamp":"1594645712602","port":9093,"version":4}
cZxid = 0x90000004a
ctime = Mon Jul 13 13:08:32 UTC 2020
mZxid = 0x90000004a
mtime = Mon Jul 13 13:08:32 UTC 2020
pZxid = 0x90000004a
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x173482eadd20008
dataLength = 250
numChildren = 0
[zk: localhost:2181(CONNECTED) 2] get /brokers/ids/1
{"listener_security_protocol_map":{"PLAINTEXT":"PLAINTEXT"},"endpoints":["PLAINTEXT://kafka-1.kafka-hs.infra.svc.cluster.local:9093"],"jmx_port":-1,"host":"kafka-1.kafka-hs.infra.svc.cluster.local","timestamp":"1594645728654","port":9093,"version":4}
cZxid = 0x900000056
ctime = Mon Jul 13 13:08:48 UTC 2020
mZxid = 0x900000056
mtime = Mon Jul 13 13:08:48 UTC 2020
pZxid = 0x900000056
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x173482eadd2000a
dataLength = 250
numChildren = 0
[zk: localhost:2181(CONNECTED) 3] get /brokers/ids/2
{"listener_security_protocol_map":{"PLAINTEXT":"PLAINTEXT"},"endpoints":["PLAINTEXT://kafka-2.kafka-hs.infra.svc.cluster.local:9093"],"jmx_port":-1,"host":"kafka-2.kafka-hs.infra.svc.cluster.local","timestamp":"1594645726723","port":9093,"version":4}
cZxid = 0x900000052
ctime = Mon Jul 13 13:08:46 UTC 2020
mZxid = 0x900000052
mtime = Mon Jul 13 13:08:46 UTC 2020
pZxid = 0x900000052
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x173482eadd20009
dataLength = 250
numChildren = 0
```

### 3.2.6 、kafka基本操作测试 

创建test01 topic

```bash
[root@manage ~]# kubectl exec -ti kafka-0 -n infra -- /opt/kafka/bin/kafka-topics.sh --create --topic test01 --zookeeper zk-0.zk-hs:2181    --partitions 3 --replication-factor 3
Created topic "test01".
```

查看topic

```bash
[root@manage ~]# kubectl exec -ti kafka-0 -n infra -- /opt/kafka/bin/kafka-topics.sh --zookeeper zk-0.zk-hs:2181 --list
test01
```

模拟生产者与消费者

生产者

```bash
[root@manage ~]# kubectl exec -ti kafka-0 -n infra -- /opt/kafka/bin/kafka-console-producer.sh --topic test01 --broker-list kafka-0.kafka-hs:9093,kafka-1.kafka-hs:9093,kafka-2.kafka-hs:9093
this is a test01 message
hell world
```

消费者

```bash
[root@manage ~]# kubectl exec -ti kafka-0 -n infra -- /opt/kafka/bin/kafka-console-consumer.sh --topic test01 --zookeeper zk-0.zk-hs:2181 --from-beginning
Using the ConsoleConsumer with old consumer is deprecated and will be removed in a future major release. Consider using the new consumer by passing [bootstrap-server] instead of [zookeeper].
this is a test01 message
hell world
```

### 3.2.7、删除topic

方法一（配置delete.topic.enable=true）
修改kafaka配置文件server.properties， 添加delete.topic.enable=true，重启kafka，之后通过kafka命令行就可以直接删除topic
  通过命令行删除topic：

```
kafka-topics.sh --delete --zookeeper {zookeeper server} --topic {topic name}
```

方法二（没有配置delete.topic.enable=true）

1、通过命令行删除topic：

```bash
kafka-topics.sh --delete --zookeeper {zookeeper server} --topic {topic name}
```

因为kafaka配置文件中server.properties没有配置delete.topic.enable=true，此时的删除并不是真正的删除，只是把topic标记为：marked for deletion
你可以通过命令,来查看所有topic：

```bash
kafka-topics --zookeeper {zookeeper server} --list 
```

2、删除kafka存储目录（server.properties文件log.dirs配置，默认为"/tmp/kafka-logs"）相关topic目录
3、 若想真正删除它，需要登录zookeeper客户端：

```bash
zkCli.sh
```

 找到topic所在的目录：

```bash
ls /brokers/topics
```

删除topic

```bash
rmr /brokers/topics/{topic name}
```

  另外被标记为marked for deletion的topic你可以在zookeeper客户端中通过命令获得：ls /admin/delete_topics/{topic name}，如果你删除了此处的topic，那么marked for deletion 标记消失

## 4.0 kafka-manager部署

```bash
mkdir kafka-manager && cd kafka-manager
```

### 4.1、定义资源清单文件

Deployment

```yaml
cat<< 'EOF' >dp.yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: kafka-manager
  namespace: infra
  labels: 
    name: kafka-manager
spec:
  replicas: 1
  selector:
    matchLabels: 
      name: kafka-manager
  template:
    metadata:
      labels: 
        app: kafka-manager
        name: kafka-manager
    spec:
      containers:
      - name: kafka-manager
        image: zenko/kafka-manager:1.3.3.22
        ports:
        - containerPort: 9000
          protocol: TCP
        env:
        - name: ZK_HOSTS
          value: zk-0.zk-hs:2181,zk-1.zk-hs:2181,zk-2.zk-hs:2181
        - name: APPLICATION_SECRET
          value: letmein
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 250m
            memory: 256Mi
      restartPolicy: Always
      terminationGracePeriodSeconds: 30
      securityContext: 
        runAsUser: 0
      schedulerName: default-scheduler
  strategy:
    type: RollingUpdate
    rollingUpdate: 
      maxUnavailable: 1
      maxSurge: 1
  revisionHistoryLimit: 7
  progressDeadlineSeconds: 600
EOF
```

Service

```yaml
cat << EOF >svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-manager
  namespace: infra
  labels:
    app: kafka-manager
spec:
  ports:
  - name: kafka
    port: 9000
    targetPort: 9000
  selector:
    app: kafka-manager
EOF
```

kafka-manager

```yaml
cat<< 'EOF' >ingress.yaml
kind: Ingress
apiVersion: extensions/v1beta1
metadata: 
  name: kafka-manager
  namespace: infra
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
  - host: km.wzxmt.com
    http:
      paths:
      - path: /
        backend: 
          serviceName: kafka-manager
          servicePort: 9000
EOF
```

### 4.2、DNS解析

```bash
km	60 IN A 10.0.0.12
```

## 4.3、创建kafka-manager

```
kubectl apply -f ./
```

### 4.4、访问kafka-manager访问界面

http://km.wzxmt.com