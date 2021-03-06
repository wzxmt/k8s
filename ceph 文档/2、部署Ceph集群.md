[toc]
# 一、Ceph版本选择

## Ceph版本来源介绍

Ceph 社区最新版本是 14，而 Ceph 12 是市面用的最广的稳定版本。
第一个 Ceph 版本是 0.1 ，要回溯到 2008 年 1 月。多年来，版本号方案一直没变，直到 2015 年 4 月 0.94.1 （ Hammer 的第一个修正版）发布后，为了避免 0.99 （以及 0.100 或 1.00 ？），制定了新策略。

x.0.z - 开发版（给早期测试者和勇士们）

x.1.z - 候选版（用于测试集群、高手们）

x.2.z - 稳定、修正版（给用户们）

x 将从 9 算起，它代表 Infernalis （ I 是第九个字母），这样第九个发布周期的第一个开发版就是 9.0.0 ；后续的开发版依次是 9.0.1 、 9.0.2 等等。
| 版本名称 | 版本号 | 发布时间 |
| ------ | ------ | ------ |
| Argonaut | 0.48版本(LTS) | 　　2012年6月3日 |
| Bobtail | 0.56版本(LTS) | 　2013年5月7日 |
| Cuttlefish | 0.61版本 | 　2013年1月1日 |
| Dumpling | 0.67版本(LTS) | 　2013年8月14日 |
| Emperor | 0.72版本 | 　　　 2013年11月9 |
| Firefly | 0.80版本(LTS) | 　2014年5月 |
| Giant | Giant | 　October 2014 - April 2015 |
| Hammer | Hammer | 　April 2015 - November 2016|
| Infernalis | Infernalis | 　November 2015 - June 2016 |
| Jewel | 10.2.9 | 　2016年4月 |
| Kraken | 11.2.1 | 　2017年10月 |
| Luminous |12.2.12  | 　2017年10月 |
| mimic | 13.2.7 | 　2018年5月 |
| nautilus | 14.2.5 | 　2019年2月 |
## Luminous新版本特性

- Bluestore
  * ceph-osd的新后端存储BlueStore已经稳定，是新创建的OSD的默认设置。
BlueStore通过直接管理物理HDD或SSD而不使用诸如XFS的中间文件系统，来管理每个OSD存储的数据，这提供了更大的性能和功能。
  * BlueStore支持Ceph存储的所有的完整的数据和元数据校验。
  * BlueStore内嵌支持使用zlib，snappy或LZ4进行压缩。（Ceph还支持zstd进行RGW压缩，但由于性能原因，不为BlueStore推荐使用zstd）
- 集群的总体可扩展性有所提高。我们已经成功测试了多达10,000个OSD的集群。
- ceph-mgr
  * ceph-mgr是一个新的后台进程，这是任何Ceph部署的必须部分。虽然当ceph-mgr停止时，IO可以继续，但是度量不会刷新，并且某些与度量相关的请求（例如，ceph df）可能会被阻止。我们建议您多部署ceph-mgr的几个实例来实现可靠性。
  * ceph-mgr守护进程daemon包括基于REST的API管理。注：API仍然是实验性质的，目前有一些限制，但未来会成为API管理的基础。
  * ceph-mgr还包括一个Prometheus插件。
  * ceph-mgr现在有一个Zabbix插件。使用zabbix_sender，它可以将集群故障事件发送到Zabbix Server主机。这样可以方便地监视Ceph群集的状态，并在发生故障时发送通知。

# 二、安装前准备
1. 安装要求 

最少三台Centos7系统虚拟机用于部署Ceph集群。硬件配置：2C4G，另外每台机器最少挂载三块硬盘(每块盘5G)  

```
cephnode01 10.0.0.61
cephnode02 10.0.0.62 
cephnode03 10.0.0.63  
```

2. 环境准备（在Ceph三台机器上操作）

（1）关闭防火墙：

```
systemctl stop firewalld
systemctl disable firewalld
```

（2）关闭selinux：

```
sed -i 's/enforcing/disabled/' /etc/selinux/config
setenforce 0
```

（3）关闭NetworkManager

```
systemctl disable NetworkManager && systemctl stop NetworkManager
```

（4）添加主机名与IP对应关系：

```
cat << EOF >>/etc/hosts
10.0.0.61 cephnode01
10.0.0.62 cephnode02
10.0.0.63 cephnode03
EOF
```

（5）设置主机名：

```
hostnamectl set-hostname cephnode01
hostnamectl set-hostname cephnode02
hostnamectl set-hostname cephnode03
```

（6）同步网络时间和修改时区

```
\cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
yum install -y chrony
systemctl enable chronyd.service --now
```

（7）设置文件描述符

```
echo "ulimit -SHn 102400" >> /etc/rc.local
cat >> /etc/security/limits.conf << EOF
1* soft nofile 65535
1* hard nofile 65535
EOF
```

（8）内核参数优化

```
cat >> /etc/sysctl.conf << EOF
kernel.pid_max = 4194303
vm.swappiness = 0
EOF
sysctl -p
```

（9）在cephnode01上配置免密登录到cephnode02、cephnode03

```
ssh-copy-id root@cephnode02
ssh-copy-id root@cephnode03
```

(10)read_ahead,通过数据预读并且记载到随机访问内存方式提高磁盘读操作

```
echo "8192" > /sys/block/sda/queue/read_ahead_kb
```

(11) I/O Scheduler，SSD要用noop，SATA/SAS使用deadline

```
echo "deadline" >/sys/block/sd[x]/queue/scheduler
echo "noop" >/sys/block/sd[x]/queue/scheduler
```

# 三、安装Ceph集群

1、编辑yum源
```
cat << 'EOF' >/etc/yum.repos.d/ceph.repo 
[Ceph]
name=Ceph packages for $basearch
baseurl=http://mirrors.163.com/ceph/rpm-nautilus/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=1

[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://mirrors.163.com/ceph/rpm-nautilus/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=1

[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.163.com/ceph/rpm-nautilus/el7/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
EOF
```
生成缓存

```
yum makecache
```

2、安装ceph-deploy

```
yum install -y ceph-deploy
```
3、创建一个my-cluster目录，所有命令在此目录下进行（文件位置和名字可以随意）
```
mkdir /my-cluster
cd /my-cluster
```
4、创建一个Ceph集群
```
ceph-deploy new cephnode01 cephnode02 cephnode03
```
5、安装Ceph软件（每个节点执行）

```
yum -y install epel-release
yum install -y ceph
```
6、生成monitor检测集群所使用的的秘钥
```
ceph-deploy mon create-initial
```
7、安装Ceph CLI，方便执行一些管理命令
```
ceph-deploy admin cephnode01 cephnode02 cephnode03
```
8、配置mgr，用于管理集群
```
ceph-deploy mgr create cephnode01 cephnode02 cephnode03
```
9、部署rgw(多个安装，负载均衡)

```
yum install -y ceph-radosgw #(每个节点执行)
ceph-deploy rgw create cephnode01
ceph-deploy rgw create cephnode02
ceph-deploy rgw create cephnode03
```
10、部署MDS（CephFS）
```
ceph-deploy mds create cephnode01 cephnode02 cephnode03 
```
创建池

```
ceph osd pool create cephfs_data 32
ceph osd pool create cephfs_metadata 32
```

创建文件系统

```
ceph fs new cephfs cephfs_metadata cephfs_data
```

查看mds状态

```
#ceph mds stat
cephfs:1 {0=node3.localdomain=up:active}
```

11、添加osd

```
ceph-deploy osd create --data /dev/sdb cephnode01
ceph-deploy osd create --data /dev/sdc cephnode01
ceph-deploy osd create --data /dev/sdd cephnode01
ceph-deploy osd create --data /dev/sdb cephnode02
ceph-deploy osd create --data /dev/sdc cephnode02
ceph-deploy osd create --data /dev/sdd cephnode02
ceph-deploy osd create --data /dev/sdb cephnode03
ceph-deploy osd create --data /dev/sdc cephnode03
ceph-deploy osd create --data /dev/sdd cephnode03
```
# 四、ceph.conf

1、该配置文件采用init文件语法，#和;为注释，ceph集群在启动的时候会按照顺序加载所有的conf配置文件。 配置文件分为以下几大块配置。

    global：全局配置。
    osd：osd专用配置，可以使用osd.N，来表示某一个OSD专用配置，N为osd的编号，如0、2、1等。
    mon：mon专用配置，也可以使用mon.A来为某一个monitor节点做专用配置，其中A为该节点的名称，ceph-monitor-2、ceph-monitor-1等。使用命令 ceph mon dump可以获取节点的名称。
    client：客户端专用配置。

2、配置文件可以从多个地方进行顺序加载，如果冲突将使用最新加载的配置，其加载顺序为。

    $CEPH_CONF环境变量
    -c 指定的位置
    /etc/ceph/ceph.conf
    ~/.ceph/ceph.conf
    ./ceph.conf

3、配置文件还可以使用一些元变量应用到配置文件，如。

    $cluster：当前集群名。
    $type：当前服务类型。
    $id：进程的标识符。
    $host：守护进程所在的主机名。
    $name：值为$type.$id。

4、ceph.conf详细参数
```
[global]#全局设置
fsid = xxxxxxxxxxxxxxx                           #集群标识ID 
mon host = 10.0.1.1,10.0.1.2,10.0.1.3            #monitor IP 地址
auth cluster required = cephx                    #集群认证
auth service required = cephx                           #服务认证
auth client required = cephx                            #客户端认证
osd pool default size = 3                             #最小副本数 默认是3
osd pool default min size = 1                           #PG 处于 degraded 状态不影响其 IO 能力,min_size是一个PG能接受IO的最小副本数
public network = 10.0.1.0/24                            #公共网络(monitorIP段) 
cluster network = 10.0.2.0/24                           #集群网络
max open files = 131072                                 #默认0#如果设置了该选项，Ceph会设置系统的max open fds
mon initial members = node1, node2, node3               #初始monitor (由创建monitor命令而定)
##############################################################
[mon]
mon data = /var/lib/ceph/mon/ceph-$id
mon clock drift allowed = 1                             #默认值0.05#monitor间的clock drift
mon osd min down reporters = 13                         #默认值1#向monitor报告down的最小OSD数
mon osd down out interval = 600      #默认值300      #标记一个OSD状态为down和out之前ceph等待的秒数
##############################################################
[osd]
osd data = /var/lib/ceph/osd/ceph-$id
osd mkfs type = xfs                                     #格式化系统类型
osd max write size = 512 #默认值90                   #OSD一次可写入的最大值(MB)
osd client message size cap = 2147483648 #默认值100    #客户端允许在内存中的最大数据(bytes)
osd deep scrub stride = 131072 #默认值524288         #在Deep Scrub时候允许读取的字节数(bytes)
osd op threads = 16 #默认值2                         #并发文件系统操作数
osd disk threads = 4 #默认值1                        #OSD密集型操作例如恢复和Scrubbing时的线程
osd map cache size = 1024 #默认值500                 #保留OSD Map的缓存(MB)
osd map cache bl size = 128 #默认值50                #OSD进程在内存中的OSD Map缓存(MB)
osd mount options xfs = "rw,noexec,nodev,noatime,nodiratime,nobarrier" #默认值rw,noatime,inode64  #Ceph OSD xfs Mount选项
osd recovery op priority = 2 #默认值10              #恢复操作优先级，取值1-63，值越高占用资源越高
osd recovery max active = 10 #默认值15              #同一时间内活跃的恢复请求数 
osd max backfills = 4  #默认值10                  #一个OSD允许的最大backfills数
osd min pg log entries = 30000 #默认值3000           #修建PGLog是保留的最大PGLog数
osd max pg log entries = 100000 #默认值10000         #修建PGLog是保留的最大PGLog数
osd mon heartbeat interval = 40 #默认值30            #OSD ping一个monitor的时间间隔（默认30s）
ms dispatch throttle bytes = 1048576000 #默认值 104857600 #等待派遣的最大消息数
objecter inflight ops = 819200 #默认值1024           #客户端流控，允许的最大未发送io请求数，超过阀值会堵塞应用io，为0表示不受限
osd op log threshold = 50 #默认值5                  #一次显示多少操作的log
osd crush chooseleaf type = 0 #默认值为1              #CRUSH规则用到chooseleaf时的bucket的类型
##############################################################
[client]
rbd cache = true #默认值 true      #RBD缓存
rbd cache size = 335544320 #默认值33554432           #RBD缓存大小(bytes)
rbd cache max dirty = 134217728 #默认值25165824      #缓存为write-back时允许的最大dirty字节数(bytes)，如果为0，使用write-through
rbd cache max dirty age = 30 #默认值1                #在被刷新到存储盘前dirty数据存在缓存的时间(seconds)
rbd cache writethrough until flush = false #默认值true  #该选项是为了兼容linux-2.6.32之前的virtio驱动，避免因为不发送flush请求，数据不回写
              #设置该参数后，librbd会以writethrough的方式执行io，直到收到第一个flush请求，才切换为writeback方式。
rbd cache max dirty object = 2 #默认值0              #最大的Object对象数，默认为0，表示通过rbd cache size计算得到，librbd默认以4MB为单位对磁盘Image进行逻辑切分
      #每个chunk对象抽象为一个Object；librbd中以Object为单位来管理缓存，增大该值可以提升性能
rbd cache target dirty = 235544320 #默认值16777216    #开始执行回写过程的脏数据大小，不能超过 rbd_cache_max_dirty
```

# 五、报错

1.ceph-deploy new cephnode01 cephnode02 cephnode03 创建集群时报错

```
[root@cephnode01 my-cluster]# ceph-deploy new cephnode01 cephnode02 cephnode03
Traceback (most recent call last):
  File "/usr/bin/ceph-deploy", line 18, in <module>
    from ceph_deploy.cli import main
  File "/usr/lib/python2.7/site-packages/ceph_deploy/cli.py", line 1, in <module>
    import pkg_resources
ImportError: No module named pkg_resources
```

解决

```
wget https://pypi.python.org/packages/ff/d4/209f4939c49e31f5524fa0027bf1c8ec3107abaf7c61fdaad704a648c281/setuptools-21.0.0.tar.gz#md5=81964fdb89534118707742e6d1a1ddb4
tar vxf setuptools-21.0.0.tar.gz 
cd setuptools-21.0.0
python setup.py  install

wget https://pypi.python.org/packages/41/27/9a8d24e1b55bd8c85e4d022da2922cb206f183e2d18fee4e320c9547e751/pip-8.1.1.tar.gz#md5=6b86f11841e89c8241d689956ba99ed7
tar vxf pip-8.1.1.tar.gz 
cd pip-8.1.1
python setup.py install

cd ../../ && rm -fr s*
```
