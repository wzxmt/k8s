## 简介

我们知道持续构建与发布是我们日常工作中必不可少的一个步骤，目前大多公司都采用 Jenkins 集群来搭建符合需求的 CI/CD 流程，然而传统的 Jenkins Slave 一主多从方式会存在一些痛点，比如：

- 主 Master 发生单点故障时，整个流程都不可用了
- 每个 Slave 的配置环境不一样，来完成不同语言的编译打包等操作，但是这些差异化的配置导致管理起来非常不方便，维护起来也是比较费劲
- 资源分配不均衡，有的 Slave 要运行的 job 出现排队等待，而有的 Slave 处于空闲状态
- 资源有浪费，每台 Slave 可能是物理机或者虚拟机，当 Slave 处于空闲状态时，也不会完全释放掉资源。

正因为上面的这些种种痛点，我们渴望一种更高效更可靠的方式来完成这个 CI/CD 流程，而 Docker 虚拟化容器技术能很好的解决这个痛点，又特别是在 Kubernetes 集群环境下面能够更好来解决上面的问题，下图是基于 Kubernetes 搭建 Jenkins 集群的简单示意图：![img](../acess/1572587229629-c79d9b57-0252-415f-9371-af63f52116c3.png)

从图上可以看到 Jenkins Master 和 Jenkins Slave 以 Pod 形式运行在 Kubernetes 集群的 Node 上，Master 运行在其中一个节点，并且将其配置数据存储到一个 Volume 上去，Slave 运行在各个节点上，并且它不是一直处于运行状态，它会按照需求动态的创建并自动删除。

这种方式的工作流程大致为：当 Jenkins Master 接受到 Build 请求时，会根据配置的 Label 动态创建一个运行在 Pod 中的 Jenkins Slave 并注册到 Master 上，当运行完 Job 后，这个 Slave 会被注销并且这个 Pod 也会自动删除，恢复到最初状态。

那么我们使用这种方式带来了哪些好处呢？

- **服务高可用**，当 Jenkins Master 出现故障时，Kubernetes 会自动创建一个新的 Jenkins Master 容器，并且将 Volume 分配给新创建的容器，保证数据不丢失，从而达到集群服务高可用。
- **动态伸缩**，合理使用资源，每次运行 Job 时，会自动创建一个 Jenkins Slave，Job 完成后，Slave 自动注销并删除容器，资源自动释放，而且 Kubernetes 会根据每个资源的使用情况，动态分配 Slave 到空闲的节点上创建，降低出现因某节点资源利用率高，还排队等待在该节点的情况。
- **扩展性好**，当 Kubernetes 集群的资源严重不足而导致 Job 排队等待时，可以很容易的添加一个 Kubernetes Node 到集群中，从而实现扩展。

是不是以前我们面临的种种问题在 Kubernetes 集群环境下面是不是都没有了啊？看上去非常完美。

## jenkins-master

#### 获取镜像

> [jenkins官网](https://jenkins.io/download/)
> [jenkins镜像](https://hub.docker.com/r/jenkins/jenkins)

下载官网上的稳定版

```bash
docker pull jenkins/jenkins:2.245-centos
docker tag jenkins/jenkins:2.245-centos harbor.wzxmt.com/infra/jenkins:latest
docker push harbor.wzxmt.com/infra/jenkins:latest
```

nfs(10.0.0.20)上创建共享资源路径

```bash
mkdir -p /data/nfs-volume/jenkins
```

#### 准备资源配置清单

RBAC

```yaml
cat << 'EOF' >rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: infra
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: jenkins
  namespace: infra
rules:
  - apiGroups: ["extensions", "apps"]
    resources: ["deployments"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create","delete","get","list","patch","update","watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get","list","watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: jenkins
  namespace: infra
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: infra
EOF
```

PVC

```yaml
cat << 'EOF' >pvc.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkins-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Delete
  nfs:
    server: 10.0.0.20
    path: /data/nfs-volume/jenkins
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: jenkins-pvc
  namespace: infra
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
EOF
```

deployment

```yaml
cat << EOF >deployment.yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: jenkins
  namespace: infra
  labels: 
    name: jenkins
spec:
  replicas: 1
  selector:
    matchLabels: 
      name: jenkins
  template:
    metadata:
      labels: 
        app: jenkins 
        name: jenkins
    spec:
      serviceAccountName: jenkins
      containers:
      - name: jenkins
        image: harbor.wzxmt.com/infra/jenkins:latest
        ports:
        - containerPort: 8080
          protocol: TCP
        - containerPort: 50000
          protocol: TCP
        env:
        - name: JAVA_OPTS
          value: -XshowSettings:vm -Dhudson.slaves.NodeProvisioner.initialDelay=0 -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85 -Duser.timezone=Asia/Shanghai
        resources:
          limits: 
            cpu: 1024m
            memory: 1Gi
          requests: 
            cpu: 1024m
            memory: 1Gi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        imagePullPolicy: IfNotPresent
        securityContext:
          runAsUser: 0     #设置以ROOT用户运行容器
          privileged: true #拥有特权
        volumeMounts:
        - name: jenkins-home
          subPath: jenkins-home
          mountPath: /var/jenkins_home
        - name: date
          mountPath: /etc/localtime
      securityContext:
        fsGroup: 1000
      volumes:
      - name: date
        hostPath:
          path: /etc/localtime
          type: ''
      - name: jenkins-home
        persistentVolumeClaim:
          claimName: jenkins-pvc
      imagePullSecrets:
      - name: harborlogin
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

svc

```yaml
cat << EOF >svc.yaml
kind: Service
apiVersion: v1
metadata: 
  name: jenkins
  namespace: infra
spec:
  ports:
  - name: web
    port: 80
    targetPort: 8080
  - name: agent 
    port: 50000
    targetPort: 50000  
  selector:
    app: jenkins
  type: ClusterIP
  sessionAffinity: None
EOF
```

ingress

```yaml
cat << EOF >ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:  
  name: jenkins
  namespace: infra
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:  
  rules:    
    - host: jenkins.wzxmt.com      
      http:        
        paths:        
        - path: /          
          backend:            
            serviceName: jenkins            
            servicePort: 80
EOF
```

**创建docker-registry**

```bash
kubectl create namespace infra
kubectl create secret docker-registry harborlogin \
--namespace=infra  \
--docker-server=https://harbor.wzxmt.com \
--docker-username=admin \
--docker-password=admin
```

#### 应用资源清单

```bash
kubectl apply -f ./
```

#### 解析域名

```
jenkins	60 IN A 10.0.0.50
```

#### 浏览器访问

[http://jenkins.wzxmt.com](http://jenkins.wzxmt.com/)

#### 优化jenkins插件下载速度

在管理机上

```bash
WORK_DIR=/data/nfs-volume/jenkins/jenkins-home
sed -i.bak 's#http://updates.jenkins-ci.org/download#https://mirrors.tuna.tsinghua.edu.cn/jenkins#g;s#http://www.google.com#https://www.baidu.com#g' ${WORK_DIR}/updates/default.json
```

从新在运算节点部署jenkins

```bash
kubectl delete -f  deployment.yaml
kubectl apply -f  deployment.yaml
```

#### 页面配置jenkins

等到服务启动成功后，我们就可以通过[http://jenkins.wzxmt.com](http://jenkins.wzxmt.com/)访问 jenkins 服务了，可以根据提示信息进行安装配置即可：![setup jenkins](../acess/image-20200622074600136.png)初始化的密码我们可以在 jenkins 的容器的日志中进行查看，也可以直接在 nfs 的共享数据目录中查看：

```shell
cat ${WORK_DIR}/secrets/initialAdminPassword
```

然后选择安装基础的插件即可。![image-20210116083928201](../acess/image-20210116083928201.png)   

安装完成后添加管理员帐号即可进入到 jenkins 主界面：

![jenkins home](../acess/image-20210116092054090.png)

#### 调整安全选项

- Manage Jenkins
  - Configure Global Security
    - Allow anonymous read access（钩上）

- Manage Jenkins
  - 防止跨站点请求伪造(取消钩)

#### 配置

接下来我们就需要来配置 Jenkins，让他能够动态的生成 Slave 的 Pod。

**第1步.** 我们需要安装**kubernetes plugin (新版本就叫 Kubernetes)**， 点击 Manage Jenkins -> Manage Plugins -> Available -> Kubernetes plugin 勾选安装即可。

![image-2020062](../acess/image-2020062208044.png)

**第2步.** 安装完毕后，点击 Manage Jenkins —> Configure System —> (拖到最下方)Add a new cloud —> 选择 Kubernetes，然后填写 Kubernetes 和 Jenkins 配置信息。![image-20200622094452898](../acess/image-20200627224756024.png)

#### 添加扩展插件

- Git Parameter 可以实现动态的从git中获取所有分支
- Config File Provider 主要可以将kubeconfig配置文件存放在jenkins里，让这个pipeline引用这个配置文件
- Extended Choice Parameter 进行对选择框插件进行扩展，可以多选，扩展参数构建，而且部署微服务还需要多选
- Blue Ocean 一个可视化、可编辑的流水线插件

#### 添加凭据

![通过jenkins交付微服务到kubernetes](https://www.linuxidc.com/upload/2020_05/200511164028156.png)
点击jenkins
![通过jenkins交付微服务到kubernetes](https://www.linuxidc.com/upload/2020_05/2005111640281526.png)
add 添加凭据
![通过jenkins交付微服务到kubernetes](https://www.linuxidc.com/upload/2020_05/2005111640281534.png)
填写harbor的用户名和密码，密码Harbor12345
描述随便写,
![通过jenkins交付微服务到kubernetes](https://www.linuxidc.com/upload/2020_05/2005111640281535.png)
再添加第二个
![通过jenkins交付微服务到kubernetes](https://www.linuxidc.com/upload/2020_05/2005111640281540.png)
git的用户名和密码
![通过jenkins交付微服务到kubernetes](https://www.linuxidc.com/upload/2020_05/2005111640281524.png)
![通过jenkins交付微服务到kubernetes](https://www.linuxidc.com/upload/2020_05/2005111640281543.png)
将这个id放到pipeline中
![通过jenkins交付微服务到kubernetes](https://www.linuxidc.com/upload/2020_05/2005111640281545.png)
将生成的密钥认证放到pipeline中

现在去添加kubeconfig的文件
![通过jenkins交付微服务到kubernetes](https://www.linuxidc.com/upload/2020_05/2005111640281518.png)
![通过jenkins交付微服务到kubernetes](https://www.linuxidc.com/upload/2020_05/2005111640281541.png)
将这个ID放到我们k8s-auth的pipeline中，这个配置文件是k8s连接kubeconfig的ID，cat /root/.kube/config 这个文件下将文件拷贝到jenkins中
![通过jenkins交付微服务到kubernetes](https://www.linuxidc.com/upload/2020_05/2005111640281529.png)

最后进行测试发布在pipeline的配置指定发布的服务进行发布

## 制作底包

根据自己需求添加相应配置

### 制作微服务底包镜像

#### Dockerfile

```bash
mkdir -p jre8 && cd jre8
cat << EOF >Dockerfile
FROM stanleyws/jre8:8u112
ADD * /opt/
RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo 'Asia/Shanghai' >/etc/timezone && \
    mkdir /opt/prom -p && mv /opt/jmx_javaagent-0.3.1.jar /opt/prom && \
    mv /opt/config.yml /opt/prom && \
    mv /opt/entrypoint.sh / && rm -f /opt/Dockerfile  
WORKDIR /opt/project_dir
CMD ["/entrypoint.sh"]
EOF
```

config.yml

```bash
cat << 'EOF' >config.yml
---
rules:
  - pattern: '.*'
EOF
```

jmx_javaagent-0.3.1.jar

```bash
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.3.1/jmx_prometheus_javaagent-0.3.1.jar -O jmx_javaagent-0.3.1.jar
```

entrypoint.sh

```bash
cat << 'EOF' >entrypoint.sh
#!/bin/sh
M_OPTS="-Duser.timezone=Asia/Shanghai -javaagent:/opt/prom/jmx_javaagent-0.3.1.jar=$(hostname -i):${M_PORT:-"12346"}:/opt/prom/config.yml"
C_OPTS=${C_OPTS}
JAR_BALL=${JAR_BALL}
exec java -jar ${M_OPTS} ${C_OPTS} ${JAR_BALL}
EOF
chmod +x entrypoint.sh
```

#### 制作镜像并推送

```bash
docker build . -t harbor.wzxmt.com/base/jre8:8u112
docker push harbor.wzxmt.com/base/jre8:8u112
```

**注意：**jre7底包制作类似(略)

### 制作Tomcat底包镜像

[Tomcat8下载链接](https://archive.apache.org/dist/tomcat/tomcat-8/v8.5.40/bin/apache-tomcat-8.5.40.tar.gz)

```bash
mkdir -p tomcat && cd tomcat 
wget https://archive.apache.org/dist/tomcat/tomcat-8/v8.5.40/bin/apache-tomcat-8.5.40.tar.gz
tar xf apache-tomcat-8.5.40.tar.gz && mv apache-tomcat-8.5.40 tomcat-8.5.40
rm -fr apache-tomcat-8.5.40.tar.gz tomcat-8.5.40/webapps/* && mkdir -p tomcat-8.5.40/webapps/ROOT
tar zcvf tomcat-8.5.40.tar.gz tomcat-8.5.40 && rm -fr tomcat-8.5.40
```

#### 简单配置tomcat

1. 关闭AJP端口

```bash
vim tomcat-8.5.40/conf/server.xml
<!-- <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" /> -->
```

1. 配置日志

- 日志级别改为INFO

```bash
vim tomcat-8.5.40/conf/logging.properties

1catalina.org.apache.juli.AsyncFileHandler.level = INFO
2localhost.org.apache.juli.AsyncFileHandler.level = INFO
java.util.logging.ConsoleHandler.level = INFO
```

- 注释掉所有关于3manager，4host-manager日志的配置

```bash
vim tomcat-8.5.40/conf/logging.properties

#3manager.org.apache.juli.AsyncFileHandler.level = FINE
#3manager.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
#3manager.org.apache.juli.AsyncFileHandler.prefix = manager.
#3manager.org.apache.juli.AsyncFileHandler.encoding = UTF-8

#4host-manager.org.apache.juli.AsyncFileHandler.level = FINE
#4host-manager.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
#4host-manager.org.apache.juli.AsyncFileHandler.prefix = host-manager.
#4host-manager.org.apache.juli.AsyncFileHandler.encoding = UTF-8
```

#### Dockerfile

```bash
cat << 'EOF' >Dockerfile
From stanleyws/jre8:8u112
ENV CATALINA_HOME /opt/tomcat-8.5.40
ENV LANG zh_CN.UTF-8
ADD * /opt/
RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \ 
    echo 'Asia/Shanghai' >/etc/timezone && mkdir -p /opt/prom && \
    mv /opt/entrypoint.sh / && rm -f /opt/Dockerfile && \
    mv /opt/jmx_javaagent-0.3.1.jar /opt/config.yml /opt/prom 
WORKDIR /opt/tomcat-8.5.40
CMD ["/entrypoint.sh"]
EOF
```

config.yml

```bash
cat << 'EOF' >config.yml
---
rules:
  - pattern: '.*'
EOF
```

jmx_javaagent-0.3.1.jar

```bash
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.3.1/jmx_prometheus_javaagent-0.3.1.jar -O jmx_javaagent-0.3.1.jar
```

entrypoint.sh (不要忘了给执行权限)

```bash
cat << 'EOF' >entrypoint.sh
#!/bin/bash
TOMCAT_VERSION=8.5.40
M_OPTS="-Duser.timezone=Asia/Shanghai -javaagent:/opt/prom/jmx_javaagent-0.3.1.jar=$(hostname -i):${M_PORT:-"12346"}:/opt/prom/config.yml"
C_OPTS=${C_OPTS}
MIN_HEAP=${MIN_HEAP:-"128m"}
MAX_HEAP=${MAX_HEAP:-"128m"}
JAVA_OPTS=${JAVA_OPTS:-"-Xmn384m -Xss256k -Duser.timezone=GMT+08  -XX:+DisableExplicitGC -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:+CMSParallelRemarkEnabled -XX:+UseCMSCompactAtFullCollection -XX:CMSFullGCsBeforeCompaction=0 -XX:+CMSClassUnloadingEnabled -XX:LargePageSizeInBytes=128m -XX:+UseFastAccessorMethods -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=80 -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+PrintClassHistogram  -Dfile.encoding=UTF8 -Dsun.jnu.encoding=UTF8"}
CATALINA_OPTS="${CATALINA_OPTS}"
JAVA_OPTS="${M_OPTS} ${C_OPTS} -Xms${MIN_HEAP} -Xmx${MAX_HEAP} ${JAVA_OPTS}"
sed -i "2c JAVA_OPTS=\"${JAVA_OPTS}\"" /opt/tomcat-${TOMCAT_VERSION}/bin/catalina.sh
sed -i "3c CATALINA_OPTS=\"${CATALINA_OPTS}\"" /opt/tomcat-${TOMCAT_VERSION}/bin/catalina.sh
/opt/tomcat-${TOMCAT_VERSION}/bin/catalina.sh run
EOF
chmod +x entrypoint.sh
```

#### 制作镜像并推送

```bash
docker build . -t harbor.wzxmt.com/base/tomcat:v8.5.40
docker push harbor.wzxmt.com/base/tomcat:v8.5.40
```

## 制作jenkins-slave镜像

#### jenkins dockerfile结构

```bash
├── Dockerfile（见下面）
├── repositories.yaml（helm chart认证文件）
├── helm（helm包管理器）
├── cert.tar.gz (harbor ca证书及私钥)
├── config.json (docker push认证)
└── apache-maven-3.6.3-bin.tar.gz（maven工具）
```

#### 下载maven修改配置

```xml-dtd
wget https://mirrors.bfsu.edu.cn/apache/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz
tar xf apache-maven-3.6.3-bin.tar.gz && mv apache-maven-3.6.3 maven-3.6.3
tar zcvf maven-3.6.3.tar.gz maven-3.6.3 && rm -rf apache-maven-3.6.3-bin.tar.gz maven-3.6.3
#修改maven-3.6.3/conf/settings.xml
...
    </mirror>
    <!--阿里云仓库-->  
    <mirror>
      <id>mirrorId</id>
      <mirrorOf>repositoryId</mirrorOf>
      <name>Nexus aliyun</name>
      <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
    </mirror>
	<!--中央仓库1-->  
    <mirror>
      <id>repo1</id>
      <mirrorOf>central</mirrorOf>
      <name>Human Readable Name for this.Mirror.</name>
      <url>http://repo1.maven.org/maven2/</url>
    </mirror>
...
```

docekr-config

```bash
cat << 'EOF' >config.json
{
	"auths": {
		"harbor.wzxmt.com": {
			"auth": "YWRtaW46YWRtaW4="
         }
    },
	"HttpHeaders": {
		"User-Agent": "Docker-Client/19.03.9 (linux)"
   }
}
EOF
```

#### 下载helm

```bash
wget https://get.helm.sh/helm-v3.4.2-linux-amd64.tar.gz
tar xf helm-v3.4.2-linux-amd64.tar.gz
mv linux-amd64/helm ./
rm -fr helm-v3.4.2-linux-amd64.tar.gz linux-amd64
```

#### 关闭helm cahrt认证

```yaml
cat << "EOF" >repositories.yaml
apiVersion: ""
generated: "0001-01-01T00:00:00Z"
repositories:
- caFile: "/opt/cert/ca.crt"
  certFile: "/opt/cert/harbor.wzxmt.com.crt"
  insecure_skip_tls_verify: false
  keyFile: "/opt/cert/harbor.wzxmt.com.key"
  name: library
  password: admin
  url: https://harbor.wzxmt.com/chartrepo/library
  username: admin
EOF
```

#### 编写dockerfile

```bash
cat << 'EOF' >Dockerfile
FROM jenkins/inbound-agent:latest
USER root
ADD * /opt/
RUN chown -R root. /opt/* && rm -f /opt/Dockerfile && \
mv /opt/helm /usr/bin/ && mv /opt/kubectl /usr/bin/ && \
mkdir -p /root/.config/helm /root/.docker && mv /opt/repositories.yaml /root/.config/helm && \
mv /opt/config.json /root/.docker
ENTRYPOINT ["jenkins-agent"]
EOF
```

#### 构建镜像

```bash
docker build . -t harbor.wzxmt.com/infra/jenkins-slave:latest
docker push harbor.wzxmt.com/infra/jenkins-slave:latest
```

## 测试jenkins

构建一次，测试jenkins连通性

```yaml
#!/usr/bin/env groovy
pipeline {
  agent {
    kubernetes {
    yaml """
apiVersion: v1
kind: Pod
metadata:
  name: jenkins-slave
  namespace: infra
spec:
  nodeName: n2
  containers:
  - name: jnlp
    image: harbor.wzxmt.com/infra/jenkins-slave:latest
    tty: true
    imagePullPolicy: Always
    volumeMounts:
      - name: docker-cmd
        mountPath: /usr/bin/docker
      - name: docker-socker
        mountPath: /run/docker.sock
      - name: date
        mountPath: /etc/localtime
      - name: jenkins-maven
        subPath: maven-cache
        mountPath: /root/.m2
  restartPolicy: Never
  imagePullSecrets:
    - name: harborlogin
  volumes:
    - name: date
      hostPath:
        path: /etc/localtime
        type: ''
    - name: docker-cmd
      hostPath:
        path: /usr/bin/docker
        type: ''
    - name: docker-socker
      hostPath:
        path: /run/docker.sock
        type: ''
    - name: jenkins-maven
      persistentVolumeClaim:
        claimName: jenkins-pvc
"""
   }
} 
stages {
      stage('test') { 
        steps {
          sh "docker ps"
        }
     }
  }
}
```

出现success，表示jenkins slave搭建成功![image-20210116091850454](../acess/image-20210116091850454.png)

## 发布dubbo-demo-service

- create new jobs

- Enter an item name

  > dubbo-demo-service

- Pipeline -> OK


Pipeline Script

```yaml
#!/usr/bin/env groovy
pipeline {
  agent {
    kubernetes {
    yaml """
apiVersion: v1
kind: Pod
metadata:
  name: jenkins
  namespace: infra
spec:
  nodeName: n2
  containers:
  - name: jnlp
    image: harbor.wzxmt.com/infra/jenkins-slave:latest
    tty: true
    imagePullPolicy: Always
    volumeMounts:
      - name: docker-cmd
        mountPath: /usr/bin/docker
      - name: docker-socker
        mountPath: /run/docker.sock
      - name: date
        mountPath: /etc/localtime
      - name: jenkins-maven
        subPath: maven-cache
        mountPath: /root/.m2
  restartPolicy: Never
  imagePullSecrets:
    - name: harborlogin
  volumes:
    - name: date
      hostPath:
        path: /etc/localtime
        type: ''
    - name: docker-cmd
      hostPath:
        path: /usr/bin/docker
        type: ''
    - name: docker-socker
      hostPath:
        path: /run/docker.sock
        type: ''
    - name: jenkins-maven
      persistentVolumeClaim:
        claimName: jenkins-pvc
"""
   }
} 
parameters {
  string defaultValue: 'dubbo-demo-service', description: 'project name. e.g: dubbo-demo-service', name: 'app_name', trim: true
  string defaultValue: 'app/dubbo-demo-service', description: 'project docker image name. e.g: app/dubbo-demo-service', name: 'image_name', trim: true
  string defaultValue: 'https://github.com/wzxmt/dubbo-demo-service.git', description: 'project git repository. e.g: https://github.com/wzxmt/dubbo-demo-service.git', name: 'git_repo', trim: true
  string defaultValue: 'master', description: 'git commit id of the project.', name: 'git_ver', trim: true
  string defaultValue: '', description: 'project docker image tag, date_timestamp recommended. e.g: 200102_0001', name: 'add_tag', trim: true
  string defaultValue: '/opt', description: 'project maven directory. e.g: ./', name: 'mvn_dir', trim: true
  string defaultValue: './dubbo-server/target', description: 'the relative path of target file such as .jar or .war package. e.g: ./dubbo-server/target', name: 'target_dir', trim: true
  string defaultValue: 'mvn clean package -Dmaven.test.skip=true', description: 'maven command. e.g: mvn clean package -e -q -Dmaven.test.skip=true', name: 'mvn_cmd', trim: true
  string defaultValue: '/opt', description: '', name: 'mvn_dir', trim: true
  choice choices: ['base/jre8:8u112', 'base/jre7:7u112'], description: 'different base images.', name: 'base_image'
  choice choices: ['maven-3.6.3', 'maven-3.6.2', 'maven-3.6.1'], description: 'different maven edition.', name: 'maven'
}
stages {
      stage('pull') { //get project code from repo 
        steps {
          sh "git clone ${params.git_repo} ${params.app_name}/${env.BUILD_NUMBER} && cd ${params.app_name}/${env.BUILD_NUMBER} && git checkout ${params.git_ver}"
        }
      }
      stage('build') { //exec mvn cmd
        steps {
          sh "cd ${params.app_name}/${env.BUILD_NUMBER} && ${params.mvn_dir}/${params.maven}/bin/${params.mvn_cmd}"
        }
      }
      stage('package') { //move jar file into project_dir
        steps {
          sh "cd ${params.app_name}/${env.BUILD_NUMBER} && cd ${params.target_dir} && mkdir project_dir && mv *.jar ./project_dir"
        }
      }
      stage('dockerfile') { //exec mvn cmd
        steps {
          writeFile file: "${params.app_name}/${env.BUILD_NUMBER}/Dockerfile", text: """FROM harbor.wzxmt.com/${params.base_image}
ADD ${params.target_dir}/project_dir /opt/project_dir"""
        }
      }
      stage('image') { //build image and push to registry
        steps {
          sh """
          cd ${params.app_name}/${env.BUILD_NUMBER} && \
          docker build -t harbor.wzxmt.com/${params.image_name}:${params.git_ver}_${params.add_tag} . && \
          docker push harbor.wzxmt.com/${params.image_name}:${params.git_ver}_${params.add_tag}
          """
         }
     }
   }
}
```

## 发布dubbo-demo-consumer

- create new jobs

- Enter an item name

  > dubbo-demo-consumer


Pipeline Script

```yaml
#!/usr/bin/env groovy
pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
metadata:
  name: jenkins
  namespace: infra
spec:
  nodeName: n2
  containers:
  - name: jnlp
    image: harbor.wzxmt.com/infra/jenkins-slave:latest
    tty: true
    imagePullPolicy: Always
    volumeMounts:
      - name: docker-cmd
        mountPath: /usr/bin/docker
      - name: docker-socker
        mountPath: /run/docker.sock
      - name: date
        mountPath: /etc/localtime
      - name: jenkins-maven
        subPath: maven-cache
        mountPath: /root/.m2
  restartPolicy: Never
  imagePullSecrets:
    - name: harborlogin
  volumes:
    - name: date
      hostPath:
        path: /etc/localtime
        type: ''
    - name: docker-cmd
      hostPath:
        path: /usr/bin/docker
        type: ''
    - name: docker-socker
      hostPath:
        path: /run/docker.sock
        type: ''
    - name: jenkins-maven
      persistentVolumeClaim:
        claimName: jenkins-pvc
"""
   }
} 
parameters {
  string defaultValue: 'dubbo-demo-consumer', description: 'project name. e.g: dubbo-demo-service', name: 'app_name', trim: true
  string defaultValue: 'app/dubbo-demo-consumer', description: 'project docker image name. e.g: app/dubbo-demo-consumer', name: 'image_name', trim: true
  string defaultValue: 'https://github.com/wzxmt/dubbo-demo-web.git', description: 'project git repository. e.g: https://github.com/wzxmt/dubbo-demo-web.git', name: 'git_repo', trim: true
  string defaultValue: 'master', description: 'git commit id of the project.', name: 'git_ver', trim: true
  string defaultValue: '', description: 'project docker image tag, date_timestamp recommended. e.g: 200102_0001', name: 'add_tag', trim: true
  string defaultValue: '/opt', description: 'project maven directory. e.g: ./', name: 'mvn_dir', trim: true
  string defaultValue: './dubbo-client/target', description: 'the relative path of target file such as .jar or .war package. e.g: ./dubbo-client/target', name: 'target_dir', trim: true
  string defaultValue: 'mvn clean package -Dmaven.test.skip=true', description: 'maven command. e.g: mvn clean package -e -q -Dmaven.test.skip=true', name: 'mvn_cmd', trim: true
  string defaultValue: '/opt', description: '', name: 'mvn_dir', trim: true
  choice choices: ['base/jre8:8u112', 'base/jre7:7u112'], description: 'different base images.', name: 'base_image'
  choice choices: ['maven-3.6.3', 'maven-3.6.2', 'maven-3.6.1'], description: 'different maven edition.', name: 'maven'
}
stages {
      stage('pull') { //get project code from repo 
        steps {
          sh "git clone ${params.git_repo} ${params.app_name}/${env.BUILD_NUMBER} && cd ${params.app_name}/${env.BUILD_NUMBER} && git checkout ${params.git_ver}"
        }
      }
      stage('build') { //exec mvn cmd
        steps {
          sh "cd ${params.app_name}/${env.BUILD_NUMBER} && ${params.mvn_dir}/${params.maven}/bin/${params.mvn_cmd}"
        }
      }
      stage('package') { //move jar file into project_dir
        steps {
          sh "cd ${params.app_name}/${env.BUILD_NUMBER} && cd ${params.target_dir} && mkdir project_dir && mv *.jar ./project_dir"
        }
      }
      stage('dockerfile') { //exec mvn cmd
        steps {
          writeFile file: "${params.app_name}/${env.BUILD_NUMBER}/Dockerfile", text: """FROM harbor.wzxmt.com/${params.base_image}
ADD ${params.target_dir}/project_dir /opt/project_dir"""
        }
      }
      stage('image') { //build image and push to registry
        steps {
          sh """
          cd ${params.app_name}/${env.BUILD_NUMBER} && \
          docker build -t harbor.wzxmt.com/${params.image_name}:${params.git_ver}_${params.add_tag} . && \
          docker push harbor.wzxmt.com/${params.image_name}:${params.git_ver}_${params.add_tag}
          """
         }
     }
   }
}
```

## 发布tomcat

- 使用admin登录

- New Item

- create new jobs

- Enter an item name

  > tomcat-demo

- Pipeline -> OK

```yaml
pipeline {
  agent {
    kubernetes {
    yaml """
apiVersion: v1
kind: Pod
metadata:
  name: jenkins
  namespace: infra
spec:
  nodeName: n2
  containers:
  - name: jnlp
    image: harbor.wzxmt.com/infra/jenkins-slave:latest
    tty: true
    imagePullPolicy: Always
    volumeMounts:
      - name: docker-cmd
        mountPath: /usr/bin/docker
      - name: docker-socker
        mountPath: /run/docker.sock
      - name: date
        mountPath: /etc/localtime
      - name: jenkins-maven
        subPath: maven-cache
        mountPath: /root/.m2
  restartPolicy: Never
  imagePullSecrets:
    - name: harborlogin
  volumes:
    - name: date
      hostPath:
        path: /etc/localtime
        type: ''
    - name: docker-cmd
      hostPath:
        path: /usr/bin/docker
        type: ''
    - name: docker-socker
      hostPath:
        path: /run/docker.sock
        type: ''
    - name: jenkins-maven
      persistentVolumeClaim:
        claimName: jenkins-pvc
"""
   }
} 
parameters {
  string defaultValue: 'dubbo-demo-web', description: 'project name. e.g: dubbo-demo-service', name: 'app_name', trim: true
  string defaultValue: 'app/dubbo-demo-web', description: 'project docker image name. e.g: app/dubbo-demo-service', name: 'image_name', trim: true
  string defaultValue: 'https://gitee.com/wzxmt/dubbo-demo-web.git', description: 'project git repository. e.g: https://gitee.com/wzxmt/dubbo-demo-web.git', name: 'git_repo', trim: true
  string defaultValue: 'apollo', description: 'git commit id of the project.', name: 'git_ver', trim: true
  string defaultValue: '', description: 'project docker image tag, date_timestamp recommended. e.g: 200102_0001', name: 'add_tag', trim: true
  string defaultValue: '/opt', description: 'project maven directory. e.g: ./', name: 'mvn_dir', trim: true
  string defaultValue: './dubbo-client/target', description: 'the relative path of target file such as .jar or .war package. e.g: ./dubbo-client/target', name: 'target_dir', trim: true
  string defaultValue: 'mvn clean package -Dmaven.test.skip=true', description: 'maven command. e.g: mvn clean package -e -q -Dmaven.test.skip=true', name: 'mvn_cmd', trim: true
  string defaultValue: '/opt', description: '', name: 'mvn_dir', trim: true
  choice choices: ['base/tomcat:v8.5.40', 'base/tomcat:v9.0.17', 'base/tomcat:v7.0.94'], description: 'project base image list in harbor.wzxmt.com.', name: 'base_image'
  choice choices: ['maven-3.6.3', 'maven-3.6.2', 'maven-3.6.1'], description: 'different maven edition.', name: 'maven'
  string defaultValue: 'ROOT', description: 'webapp dir.', name: 'webapp_DIR', trim: true
}
stages {
      stage('pull') { //get project code from repo 
        steps {
          sh "git clone ${params.git_repo} ${params.app_name}/${env.BUILD_NUMBER} && cd ${params.app_name}/${env.BUILD_NUMBER} && git checkout ${params.git_ver}"
        }
      }
      stage('build') { //exec mvn cmd
        steps {
          sh "cd ${params.app_name}/${env.BUILD_NUMBER} && ${params.mvn_dir}/${params.maven}/bin/${params.mvn_cmd}"
        }
      }
      stage('package') { //move jar file into project_dir
        steps {
          sh "cd ${params.app_name}/${env.BUILD_NUMBER} && cd ${params.target_dir} && mkdir project_dir && mv *.jar ./project_dir"
        }
      }
      stage('image') { //build image and push to registry
        steps {
          writeFile file: "${params.app_name}/${env.BUILD_NUMBER}/Dockerfile", text: """FROM harbor.wzxmt.com/${params.base_image}
ADD ${params.target_dir}/project_dir /opt/project_dir"""
          sh "cd  ${params.app_name}/${env.BUILD_NUMBER} && docker build -t harbor.wzxmt.com/${params.image_name}:${params.git_ver}_${params.add_tag} . && docker push harbor.wzxmt.com/${params.image_name}:${params.git_ver}_${params.add_tag}"
        }
      }
   }
}
```

## 部署

dubbo-demo-service

```yaml
cat << 'EOF' >dubbo-demo-service.yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: dubbo-demo-service
  namespace: app
  labels: 
    name: dubbo-demo-service
spec:
  replicas: 1
  selector:
    matchLabels: 
      name: dubbo-demo-service
  template:
    metadata:
      labels: 
        app: dubbo-demo-service
        name: dubbo-demo-service
    spec:
      containers:
      - name: dubbo-demo-service
        image: harbor.wzxmt.com/app/dubbo-demo-service:master_20210603
        ports:
        - containerPort: 20880
          protocol: TCP
        env:
        - name: JAR_BALL
          value: dubbo-server.jar
        imagePullPolicy: IfNotPresent
      imagePullSecrets:
      - name: harborlogin
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

dubbo-demo-consumer

```yaml
cat << 'EOF' >dubbo-demo-consumer.yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: dubbo-demo-consumer
  namespace: app
  labels: 
    name: dubbo-demo-consumer
spec:
  replicas: 1
  selector:
    matchLabels: 
      name: dubbo-demo-consumer
  template:
    metadata:
      labels: 
        app: dubbo-demo-consumer
        name: dubbo-demo-consumer
    spec:
      containers:
      - name: dubbo-demo-consumer
        image: harbor.wzxmt.com/app/dubbo-demo-consumer:master_20210603
        ports:
        - containerPort: 8080
          protocol: TCP
        - containerPort: 20880
          protocol: TCP
        env:
        - name: JAR_BALL
          value: dubbo-client.jar
        imagePullPolicy: IfNotPresent
      imagePullSecrets:
      - name: harborlogin
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
---
kind: Service
apiVersion: v1
metadata: 
  name: dubbo-demo-consumer
  namespace: app
spec:
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
  selector: 
    app: dubbo-demo-consumer
  clusterIP: None
  type: ClusterIP
  sessionAffinity: None
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: dubbo-demo-consumer
  namespace: app
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`demo-test.wzxmt.com`) && PathPrefix(`/`)
    kind: Rule
    services:
    - name: dubbo-demo-consumer
      port: 8080
EOF
```

http://demo-test.wzxmt.com/hello?name=wzxmt