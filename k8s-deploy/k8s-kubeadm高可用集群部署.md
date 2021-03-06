## 1. 准备服务器

| 系统类型   | IP地址     | 节点角色 | CPU  | Memory | Hostname |
| :--------- | ---------- | -------- | ---- | ------ | -------- |
| centos7-64 | 10.0.0.150 | k8s-vip  | 1    | 2G     |          |
| centos7-64 | 10.0.0.31  | master01 | 1    | 2G     | master01 |
| centos7-64 | 10.0.0.32  | master02 | 1    | 2G     | master02 |
| centos7-64 | 10.0.0.33  | node01   | 1    | 2G     | node01   |
| centos7-64 | 10.0.0.34  | node01   | 1    | 2G     | node02   |
| centos7-64 | 10.0.0.35  | node01   | 1    | 2G     | node03   |

## 2. 系统设置(所有节点）

#### 2.1 配置hosts文件

配置host，使每个Node都可以通过名字解析到ip地址

```bash
cat<< EOF >>/etc/hosts
10.0.0.31 master01
10.0.0.32 master02
10.0.0.33 node01
10.0.0.34 node02
10.0.0.35 node03
EOF
```

#### 2.2 关闭、禁用防火墙,关闭selinux与swap(让所有机器之间都可以通过任意端口建立连接)

**关闭防火墙**

```bash
firewall-cmd --state
systemctl stop firewalld.service
systemctl disable firewalld.service
```

**关闭selinux**

```bash
setenforce 0
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

**关闭swap**

```bash
swapoff -a
echo "vm.swappiness = 0" >>/etc/sysctl.d/kubernetes.conf
sed -i 's/.*swap.*/#&/' /etc/fstab
```

因为这里本次用于测试两台主机上还运行其他服务，关闭swap可能会对其他服务产生影响，所以这里修改kubelet的配置去掉这个限制。 使用kubelet的启动参数--fail-swap-on=false去掉必须关闭swap的限制，修改vim /etc/sysconfig/kubelet，加入：

```bash
cat<< EOF >/etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS=--fail-swap-on=false
EOF
```

#### 2.3 更新 yum 源

```bash
cd /etc/yum.repos.d
mv CentOS-Base.repo CentOS-Base.repo.bak
mv epel.repo  epel.repo.bak
curl https://mirrors.aliyun.com/repo/Centos-7.repo -o CentOS-Base.repo 
curl https://mirrors.aliyun.com/repo/epel-7.repo -o epel.repo
cd -
```

#### 2.4 设置系统参数 - 允许路由转发，不对bridge的数据进行处理

```bash
#写入配置文件
cat <<EOF > /etc/sysctl.d/kubernetes.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
#生效配置文件
modprobe br_netfilter
```

#### 2.5 最终内核参数优化

```bash
cat << EOF >/etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0 # 禁止使用 swap 空间，只有当系统 OOM 时才允许使用它
vm.overcommit_memory=1 # 不检查物理内存是否够用
vm.panic_on_oom=0 # 开启 OOM
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF
```

加载内核

```
sysctl -p /etc/sysctl.d/kubernetes.conf
```

#### 2.6 kube-proxy开启ipvs的前置条件

由于ipvs已经加入到了内核的主干，所以为kube-proxy开启ipvs的前提需要加载以下的内核模块：

```bash
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules
bash /etc/sysconfig/modules/ipvs.modules
lsmod|egrep "ip_vs|nf_conntrack_ipv4"
```

上面脚本创建了的/etc/sysconfig/modules/ipvs.modules文件，保证在节点重启后能自动加载所需模块。

各个节点上已经安装了ipset软件包与管理工具

```bash
yum install -y ipset ipvsadm
```

如果以上前提条件如果不满足，则即使kube-proxy的配置开启了ipvs模式，也会退回到iptables模式。

#### 2.7 调整系统时区

```bash
# 设置系统时区为中国/上海
timedatectl set-timezone Asia/Shanghai
# 将当前的 UTC 时间写入硬件时钟timedatectl set-local-rtc 0
# 重启依赖于系统时间的服务systemctl restart rsyslog
systemctl restart crond
```

#### 2.8 设置 rsyslogd 和 systemd journald

```bash
mkdir /var/log/journal # 持久化保存日志的目录
mkdir /etc/systemd/journald.conf.d
cat << EOF >/etc/systemd/journald.conf.d/99-prophet.conf
[Journal]
# 持久化保存到磁盘
Storage=persistent
# 压缩历史日志
Compress=yes
SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000
# 最大占用空间 10G
SystemMaxUse=10G
# 单日志文件最大 200M
SystemMaxFileSize=200M
# 日志保存时间 2 周
MaxRetentionSec=2week
# 不将日志转发到 
syslogForwardToSyslog=no
EOF
systemctl restart systemd-journald
```

#### 2.9 升级系统内核为 4.44

CentOS 7.x 系统自带的 3.10.x 内核存在一些 Bugs，导致运行的 Docker、Kubernetes 不稳定 

```bash
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
```

 安装完成后检查 /boot/grub2/grub.cfg 中对应内核 menuentry 中是否包含 initrd16 配置，如果没有，再安装一次！

```bash
yum --enablerepo=elrepo-kernel install -y kernel-lt
```

设置开机从新内核启动

```bash
grub2-set-default 0
```

查看内核

```bash
uname -r
4.4.218-1.el7.elrepo.x86_64
```

#### 2.30 关闭 NUMA

备份grub

```bash
cp /etc/default/grub{,.bak}
```

在 GRUB_CMDLINE_LINUX 一行添加 `numa=off` 参数，如下所示：

```bash
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rhgb quiet numa=off"
```

加载内核

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```

## 3. 安装docker(node节点）

### 3.1 yum安装

#### 3.1.1 安装依赖

```bash
yum install -y yum-utils device-mapper-persistent-data lvm2
```

#### 3.1.2 配置docker源

```bash
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast
rpm --import https://mirrors.aliyun.com/docker-ce/linux/centos/gpg
```

#### 3.1.3 查看docker版号

```bash
yum list docker-ce.x86_64  --showduplicates |sort -r
```

 Kubernetes 1.16当前支持的docker版本列表是 Docker版本1.13.1、17.03、17.06、17.09、18.06、18.09 

#### 3.1.4 安装 docker

```bash
yum makecache fast
yum install -y docker-ce-18.09.9-3.el7 
```

### 3.2 二进制安装

#### 3.2.1 下载地址

```bash
https://download.docker.com/linux/static/stable/x86_64/
```

#### 3.2.2 解压安装

```bash
tar zxvf docker-18.09.6.tgz
mv docker/* /usr/bin
mkdir /etc/docker
mkdir -p /data/docker
mv daemon.json /etc/docker
mv docker.service /usr/lib/systemd/system
```

#### 3.2.3 systemctl管理docker

```bash
cat > /usr/lib/systemd/system/docker.service << EOF
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target firewalld.service
[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
[Install]
WantedBy=multi-user.target
EOF
```

### 3.3 配置所有ip的数据包转发

```bash
#找到ExecStart=xxx，在这行下面加入一行，内容如下：(k8s的网络需要)
sed -i.bak '/ExecStart/a\ExecStartPost=/sbin/iptables -I FORWARD -s 0.0.0.0/0 -j ACCEPT' /lib/systemd/system/docker.service
```

### 3.4 docker基础优化

```bash
mkdir -p /etc/docker/
cat << EOF >/etc/docker/daemon.json
{
    "log-driver": "json-file",
    "graph":"/data/docker",
    "log-opts": { "max-size": "100m"},
    "exec-opts": ["native.cgroupdriver=systemd"],
    "registry-mirrors": ["https://s3w3uu4l.mirror.aliyuncs.com"],
    "max-concurrent-downloads": 10,
    "max-concurrent-uploads": 10,
    "storage-driver": "overlay2", 
    "storage-opts": ["overlay2.override_kernel_check=true"]
}
EOF
```

### 3.5 启动服务

```bash
#设置 docker 开机服务启动
systemctl enable --now docker.service 
```

## 4. apiserver高可用部署

#### 4.1 master节点全部ssh免密登录

master01上操作

```bash
#生成密钥
ssh-keygen -t dsa -f "/root/.ssh/id_dsa" -N "" -q
#将公钥作为认证信息
cat /root/.ssh/id_dsa.pub >.ssh/authorized_keys
scp -r .ssh master02:/root/
```

#### 4.2 两台master节点安装keepalived,haproxy

```bash
yum install -y keepalived haproxy
systemctl enable keepalived.service haproxy.service
```

#### 4.3 keepalived配置及启动

keepalived配置文件，其余节点修改state为BACKUP，priority小于主节点即可；

```bash
cat<< EOF >/etc/keepalived/keepalived.conf
! Configuration File for keepalived
vrrp_script chk_apiserver {
        script "/etc/keepalived/check_apiserver.sh"
        interval 4
        weight 60  
}
vrrp_instance VI_1 {
    state MASTER  
    interface eth0
    virtual_router_id 51
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    track_script {
        chk_apiserver
    }
    virtual_ipaddress {
        10.0.0.150
    }
}
EOF
```

check_apiserver.sh

```bash
cat<< 'EOF' >/etc/keepalived/check_apiserver.sh
#!/bin/bash
flag=$(ps -ef|grep -v grep|grep -w 'kube-apiserver' &>/dev/null;echo $?)
if [[ $flag != 0 ]];then
        echo "kube-apiserver is down,close the keepalived"
        #systemctl stop keepalived
fi
EOF
chmod +x /etc/keepalived/check_apiserver.sh
```

 启动keepalived

```bash
systemctl restart keepalived.service
journalctl -f -u keepalived.service
```

 **测试一下keepalived虚拟ip是否启动成功** 

分别在master01和master02执行如下命令，可以发现只有master01有结果

```bash
ip a | grep 10.0.0.150
inet 10.0.0.150/32 scope global eth0
```

说明目前10.0.0.150虚拟ip是漂移到master01机器上，因为master01的keepalived设置为MASTER

**测试keepalived的ip漂移功能**

尝试重启master01机器，重启期间，虚拟ip会漂移到master02机器，等master01重启完毕，虚拟ip会重新漂移到master01上,证明keepalived运行正常

#### 4.4 haproxy配置及启动

 在master01和master02节点上，分别配置haproxy ：

```bash
cat > /etc/haproxy/haproxy.cfg << EOF 
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
#---------------------------------------------------------------------
listen stats
    mode   http
    bind :10086
    stats   enable
    stats realm Haproxy\ Statistics
    stats   uri     /admin?stats
    stats   auth    admin:admin
    stats   admin   if TRUE
#---------------------------------------------------------------------
frontend   kube-apiserver 
   bind *:8443
   mode tcp
   default_backend             apiserver
#---------------------------------------------------------------------
backend apiserver
    balance     roundrobin
    mode tcp
    server  master01 10.0.0.31:6443 check weight 1 maxconn 2000 check inter 2000 rise 2 fall 3
    server  master02 10.0.0.32:6443 check weight 1 maxconn 2000 check inter 2000 rise 2 fall 3
    server  master02 10.0.0.33:6443 check weight 1 maxconn 2000 check inter 2000 rise 2 fall 3
EOF
```

####  4.5 启动haproxy

```bash
systemctl restart haproxy.service
journalctl -f -u haproxy.service
```

## 5. 安装kubeadm、kubelet和kubectl

#### 5.1 配置kubernetes源为阿里的yum源

```bash
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

#### 5.2  查看kubeadm、kubelet和kubectl版号

```bash
yum makecache fast
yum list kubeadm kubelet kubectl  --showduplicates
```

#### 5.2  各节点安装kubernetes

```bash
#master
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet 
#node
yum install -y kubelet kubeadm
systemctl enable kubelet
```

Kubelet负责与其他节点集群通信，并进行本节点Pod和容器生命周期的管理。
Kubeadm是Kubernetes的自动化部署工具，降低了部署难度，提高效率。
Kubectl是Kubernetes集群管理工具。
注：因为kubeadm默认生成的证书有效期只有一年，所以kubeadm等安装成功后，可以用编译好的kubeadm替换掉默认的kubeadm。后面初始化k8s生成的证书都是100年。

## 6.  初始化master

#### 6.1 始化master01

初始化之前检查haproxy是否正在运行，keepalived是否正常运作 ：

 查看所需的镜像 

```bash
kubeadm config images list
#所需镜像
k8s.gcr.io/kube-apiserver:v1.16.3
k8s.gcr.io/kube-controller-manager:v1.16.3
k8s.gcr.io/kube-scheduler:v1.16.3
k8s.gcr.io/kube-proxy:v1.16.3
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.15-0
k8s.gcr.io/coredns:1.6.2
```

 因为k8s.gcr.io地址在国内是不能访问的，能上外网可以通过以下命令提前把镜像pull下来 

```
kubeadm config images pull
```

因为不能翻墙，这有三种种解决办法：

##### 6.11 根据需要的版本，直接拉取国内镜像，并修改tag

```bash
cat << 'EOF' >kubeadm_get_images.sh
#!/bin/bash
## 使用如下脚本下载国内镜像，并修改tag为google的tag
set -e
KUBE_VERSION=v1.16.3
KUBE_PAUSE_VERSION=3.1
ETCD_VERSION=3.3.15-0
CORE_DNS_VERSION=1.6.2

GCR_URL=k8s.gcr.io
ALIYUN_URL=registry.cn-hangzhou.aliyuncs.com/google_containers

images=(kube-proxy:${KUBE_VERSION}
kube-scheduler:${KUBE_VERSION}
kube-controller-manager:${KUBE_VERSION}
kube-apiserver:${KUBE_VERSION}
pause:${KUBE_PAUSE_VERSION}
etcd:${ETCD_VERSION}
coredns:${CORE_DNS_VERSION})

for imageName in ${images[@]} ; do
  docker pull $ALIYUN_URL/$imageName
  docker tag  $ALIYUN_URL/$imageName $GCR_URL/$imageName
  docker rmi $ALIYUN_URL/$imageName
done
EOF
sh kubeadm_get_images.sh
```

生成初始化配置文件

```bash
cat << EOF >kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: v1.16.3 # 指定版本
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
controlPlaneEndpoint: "10.0.0.150:8443" # haproxy地址及端口
networking:
  dnsDomain: cluster.local
  serviceSubnet:  10.1.0.0/16 #用于指定SVC的网络范围；
  podSubnet: 10.244.0.0/16 # 计划使用flannel网络插件，指定pod网段及掩码
EOF
```

然后进行初始化

```
kubeadm init --config=kubeadm-config.yaml --upload-certs
```

##### 6.12.在初始化时指定使用国内镜像库：registry.aliyuncs.com/google_containers (推荐使用）

生成初始化配置文件

```bash
cat << EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: v1.16.3 # 指定版本
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
controlPlaneEndpoint: "10.0.0.150:8443" # haproxy地址及端口
networking:
  dnsDomain: cluster.local
  serviceSubnet:  10.1.0.0/16 #用于指定SVC的网络范围；
  podSubnet: 10.244.0.0/16 # 计划使用flannel网络插件，指定pod网段及掩码
EOF
```

开始初始化

```bash
kubeadm init --config=kubeadm-config.yaml --upload-certs
```

##### 6.13直接在命令行指定相应配置(只支持1.16.0以上）

```bash
kubeadm init --kubernetes-version=1.18.2 \
--apiserver-advertise-address=10.0.0.31 \
--image-repository registry.aliyuncs.com/google_containers \
--control-plane-endpoint 10.0.0.150:8443 \
--service-cidr=10.1.0.0/16 \
--pod-network-cidr=10.244.0.0/16 \
--upload-certs
```

##### 6.14初始化成功后，会看到大概如下提示，下面信息先保留。后续添加master节点，添加node节点需要用到

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 10.0.0.150:8443 --token twg8nt.a8ofqr4cddvwtzml \
    --discovery-token-ca-cert-hash sha256:0822b67f493fea80e72e967d74885fb098e5c7c506a975ebd905c302e8515cf4 \
    --control-plane --certificate-key d976c6198bd13ef7dc57de062bb844946cec21f336c4ee0abe0f90cadc570eb6

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.0.0.150:8443 --token twg8nt.a8ofqr4cddvwtzml \
    --discovery-token-ca-cert-hash sha256:0822b67f493fea80e72e967d74885fb098e5c7c506a975ebd905c302e8515cf4
```

上面记录了完成的初始化输出的内容，根据输出的内容基本上可以看出手动初始化安装一个Kubernetes集群所需要的关键步骤。 其中有以下关键内容：

- [kubelet-start] 生成kubelet的配置文件”/var/lib/kubelet/config.yaml”
- [certs]生成相关的各种证书
- [kubeconfig]生成相关的kubeconfig文件
- [control-plane]使用/etc/kubernetes/manifests目录中的yaml文件创建apiserver、controller-manager、scheduler的静态pod
- [bootstraptoken]生成token记录下来，后边使用kubeadm join往集群中添加节点时会用到
- 下面的命令是配置常规用户如何使用kubectl访问集群：

按提示执行如下命令，kubectl就能使用了

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

如果集群初始化遇到问题，可以使用下面的命令进行清理：

```bash
kubeadm reset
ifconfig cni0 down
ip link delete cni0
ifconfig flannel.1 down
ip link delete flannel.1
rm -rf /var/lib/cni/
```

####  6.2 安装Pod Network

获取所有pod状态

```bash
[root@master01 ~]# kubectl get pods -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-58cc8c89f4-c5t99           0/1     Pending   0          22m
coredns-58cc8c89f4-jv9p4           0/1     Pending   0          22m
etcd-master01                      1/1     Running   1          21m
kube-apiserver-master01            1/1     Running   1          21m
kube-controller-manager-master01   1/1     Running   1          21m
kube-proxy-gbnft                   1/1     Running   1          22m
kube-scheduler-master01            1/1     Running   1          21m
```

coredns处于Pending状态，journalctl -f -u kubelet.service日志

```bash
Nov 13 20:32:20 master01 kubelet[7791]: W1113 20:32:20.055543    7791 cni.go:237] Unable to update cni config: no networks found in /etc/cni/net.d
Nov 13 20:32:23 master01 kubelet[7791]: E1113 20:32:23.440739    7791 kubelet.go:2187] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```

这是因为没安装网络，首先把kube-flannel镜像拉下来(所有节点)

```bash
cat << 'EOF' >flannel.sh
#!/bin/bash
set -e
FLANNEL_VERSION=v0.11.0
 
# 在这里修改源
QUAY_URL=quay.io/coreos
QINIU_URL=registry.cn-hangzhou.aliyuncs.com/mirror-suke
 
images=(flannel:${FLANNEL_VERSION}-amd64)
 
for imageName in ${images[@]} ; do
  docker pull $QINIU_URL/$imageName
  docker tag  $QINIU_URL/$imageName $QUAY_URL/$imageName
  docker rmi $QINIU_URL/$imageName
done
EOF
sh flannel.sh
```

创建kube-flannel pod

网络不好，可以先下下来，再执行

```bash
kubectl apply -f kube-flannel.yml
kubectl apply -f kube-flannel-rbac.yml
```

**kube-flannel**

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

**kube-flannel-rbac**

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
```

如果Node有多个网卡的话，参考[flannel issues 39701](https://github.com/kubernetes/kubernetes/issues/39701)，目前需要在kube-flannel.yml中使用–iface参数指定集群主机内网网卡的名称，否则可能会出现dns无法解析。需要将kube-flannel.yml下载到本地，flanneld启动参数加上–iface=<iface-name>

```bash
containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.11.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=eth1
......
```

确保所有的Pod都处于Running状态

```bash
[root@master01 ~]# kubectl get pods -n kube-system
NAME                               READY   STATUS    RESTARTS   AGE
coredns-58cc8c89f4-dsdlc           1/1     Running   0          45m
coredns-58cc8c89f4-w9plv           1/1     Running   0          45m
etcd-master01                      1/1     Running   0          44m
kube-apiserver-master01            1/1     Running   0          44m
kube-controller-manager-master01   1/1     Running   0          44m
kube-flannel-ds-amd64-mf66l        1/1     Running   0          5m14s
kube-proxy-76ldd                   1/1     Running   0          45m
kube-scheduler-master01            1/1     Running   0          44m
```

查看一下集群状态，确认个组件都处于healthy状态：

```bash
[root@master01 ~]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true"}
```

#### 6.3 测试集群DNS是否可用

```bash
kubectl run curl --rm --image=radial/busyboxplus:curl -it
```

**发现创建的pod节点一直处于Pending状态，这时候需要去污(重要)**

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

进入后执行nslookup kubernetes.default确认解析正常:

```bash
nslookup kubernetes.default

Server:    10.1.0.10
Address 1: 10.1.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default
Address 1: 10.1.0.1 kubernetes.default.svc.cluster.local
```

退出容器，保持运行：ctrl Q +P
进入容器：

```bash
kubectl attach curl-6bf6db5c4f-n4txt -c curl -i -t
```

测试OK后，删除掉curl这个Pod

```bash
kubectl delete deploy curl
```

#### 6.4 Kubernetes集群中添加Node节点

默认token的有效期为24小时，当过期之后，该token就不可用了，以后加入节点需要新token

 master重新生成新的token

```bash
[root@master01 ~]# kubeadm token create
tkxyys.8ilumwddiexjd8g2

[root@master01 ~]# kubeadm token list
TOKEN                     TTL       EXPIRES                     USAGES                   DESCRIPTION   EXTRA GROUPS
tkxyys.8ilumwddiexjd8g2   23h       2019-07-10T21:19:17+08:00   authentication,signing   <none>        system:bootstrappers:kubeadm:default-node-token
```

获取ca证书`sha256`编码hash值

```bash
[root@master ~]# openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt|openssl rsa -pubin -outform der 2>/dev/null|openssl dgst -sha256 -hex|awk '{print $NF}'
2e4ec2c6267389ccc2aa293a61ab474b0304778d56dfb07f5105a709d3b798e6
```

添加node节点

```bash
kubeadm join 10.0.0.150:8443 --token 4qcl2f.gtl3h8e5kjltuo0r \
--discovery-token-ca-cert-hash sha256:7ed5404175cc0bf18dbfe53f19d4a35b1e3d40c19b10924275868ebf2a3bbe6e \
--ignore-preflight-errors=all
```

node01加入集群很是顺利，下面在master节点上执行命令查看集群中的节点：

```bash
[root@master01 ~]# kubectl get node
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   18m   v1.15.0
node01 <none>   master   11m   v1.15.0
```

节点没有ready 一般是由于flannel 插件没有装好，可以通过查看kube-system 的pod 验证

#### 6.5 如何从集群中移除Node

如果需要从集群中移除node01这个Node执行下面的命令：

在master节点上执行：

```bash
kubectl drain node01 --delete-local-data --force --ignore-daemonsets
```

在node01上执行：

```bash
kubeadm reset
ifconfig cni0 down
ip link delete cni0
ifconfig flannel.1 down
ip link delete flannel.1
rm -rf /var/lib/cni/
```

在master上执行：

```bash
kubectl delete node node01
```

不在master节点上操作集群，而是在其他工作节点上操作集群:

需要将master节点上面的kubernetes配置文件拷贝到当前节点上，然后执行kubectl命令:

```bash
#将主配置拉取到本地scp root@node01:/etc/kubernetes/admin.conf /etc/kubernetes/
#常规用户如何使用kubectl访问集群配置mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 6.6 添加master

使用之前生成的提示信息，在master02执行，就能添加一个master

```bash
kubeadm join 10.0.0.150:8443 --token 66jnza.vwvw9xp6hwkwmrtz \
    --discovery-token-ca-cert-hash sha256:46c600824f2a5c4c29ba3f0b5667c3728604ab64a40d880de8f89eaceb9b6531 \
    --experimental-control-plane --certificate-key 0a050aa3d2ce1b9366a66d5fe01946fa249340dd5088b07a05d37f465ed41150
```


执行完如上初始化命令，第二个master节点就能添加到集群

提示信息里面有如下内容，应该是之前指定参数上传的证书2个小时后会被删除，可以使用命令重新上传证书

```bash
Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use 
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.
```

#### 6.7 kube-proxy开启ipvs

修改ConfigMap的kube-system/kube-proxy中的config.conf，mode: “ipvs”

```bash
kubectl edit cm kube-proxy -n kube-system
```

重启各个节点上的kube-proxy pod：

```bash
kubectl get pod -n kube-system | grep kube-proxy | awk '{system("kubectl delete pod "$1" -n kube-system")}'
```

查看kube-proxy pod

```bash
kubectl get pod -n kube-system | grep kube-proxy

kube-proxy-62jb4                       1/1     Running   0          2m54s
kube-proxy-7k4bc                       1/1     Running   0          2m13s
kube-proxy-hrs9n                       1/1     Running   0          2m29s
kube-proxy-jk85p                       1/1     Running   0          3m17s
kube-proxy-lpdsp                       1/1     Running   0          2m45s
```

查看kube-proxy日志

```bash
 kubectl logs `kubectl get pod -n kube-system | grep kube-proxy|awk 'NR==1{print $1}'` -n kube-system
 
I1106 15:33:46.141633       1 server_others.go:170] Using ipvs Proxier.
W1106 15:33:46.142109       1 proxier.go:401] IPVS scheduler not specified, use rr by default
I1106 15:33:46.142408       1 server.go:534] Version: v1.15.0
I1106 15:33:46.161386       1 conntrack.go:52] Setting nf_conntrack_max to 131072
I1106 15:33:46.162251       1 config.go:187] Starting service config controller
I1106 15:33:46.162278       1 controller_utils.go:1029] Waiting for caches to sync for service config controller
I1106 15:33:46.162300       1 config.go:96] Starting endpoints config controller
I1106 15:33:46.162324       1 controller_utils.go:1029] Waiting for caches to sync for endpoints config controller
I1106 15:33:46.266148       1 controller_utils.go:1036] Caches are synced for endpoints config controller
I1106 15:33:46.463107       1 controller_utils.go:1036] Caches are synced for service config controller
```

日志中打印出了Using ipvs Proxier，说明ipvs模式已经开启。

## 7.  报错解决

1、Kubernetes报错Failed to get system container stats for "/system.slice/kubelet.service"

 在kubelet配置文件10-kubeadm.conf中的 Environment中添加"--runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice"

```bash
sed -ri.bak "s#(.*)(kubelet\.conf)#\1\2 --runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice#g" /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
```

然后重启

```bash
systemctl daemon-reload
systemctl restart kubelet.service
journalctl -f -u kubelet.service
```

2、清除退出容器

```bash
docker rm `docker ps -a|grep 'Exited'|awk '{print $1}'`
```

到此，k8s集群部署完毕

