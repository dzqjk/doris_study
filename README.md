## 一、Doris简介

### 1.1、Doris 概述

~~~properties
	Apache Doris 由百度大数据部研发（之前叫百度 Palo，2018 年贡献到 Apache 社区后，
更名为 Doris ），在百度内部，有超过 200 个产品线在使用，部署机器超过 1000 台，单一
业务最大可达到上百 TB。
	Apache Doris 是一个现代化的 MPP（Massively Parallel Processing，即大规模并行处理）
分析型数据库产品。仅需亚秒级响应时间即可获得查询结果，有效地支持实时数据分析。
	Apache Doris 的分布式架构非常简洁，易于运维，并且可以支持 10PB 以上的超大数据集。
	Apache Doris 可以满足多种数据分析需求，例如固定历史报表，实时数据分析，交互式
数据分析和探索式数据分析等。
~~~

![image-20231119130243025](images/doris-study/image-20231119130243025.png)

![image-20231119130300094](images/doris-study/image-20231119130300094.png)

### 1.2、Doris 架构

![image-20231119125826416](images/doris-study/image-20231119125826416.png)

~~~
	Doris 的架构很简洁，只设 FE(Frontend)、BE(Backend)两种角色、两个进程，不依赖于 外部组件，方便部署和运维，FE、BE 都可线性扩展。
⚫ FE（Frontend）：存储、维护集群元数据；负责接收、解析查询请求，规划查询计划， 调度查询执行，返回查询结果。主要有三个角色： 
	（1）Leader 和 Follower：主要是用来达到元数据的高可用，保证单节点宕机的情况下， 元数据能够实时地在线恢复，而不影响整个服务。 
	（2）Observer：用来扩展查询节点，同时起到元数据备份的作用。如果在发现集群压力 非常大的情况下，需要去扩展整个查询的能力，那么可以加 observer 的节点。observer 不 参与任何的写入，只参与读取。
⚫ BE（Backend）：负责物理数据的存储和计算；依据 FE 生成的物理计划，分布式地执 行查询。
	数据的可靠性由 BE 保证，BE 会对整个数据存储多副本或者是三副本。副本数可根据 需求动态调整。
⚫ MySQL Client 
	Doris 借助 MySQL 协议，用户使用任意 MySQL 的 ODBC/JDBC 以及 MySQL 的客户 端，都可以直接访问 Doris。
⚫ Broker
	Broker 为一个独立的无状态进程。封装了文件系统接口，提供 Doris 读取远端存储系统 中文件的能力，包括 HDFS，S3，BOS 等。
~~~

## 二、编译与安装

~~~
安装 Doris，需要先通过源码编译，主要有两种方式：使用 Docker 开发镜像编译（推荐）、直接编译。
直接编译的方式，可以参考官网：https://doris.apache.org/zh-CN/installing/compilation.html
~~~

### 2.1、安装 Docker 环境

~~~shell
## 1）Docker 要求 CentOS 系统的内核版本高于 3.10 ，首先查看系统内核版本是否满足
uname -r
## 2）使用 root 权限登录系统，确保 yum 包更新到最新
sudo yum update -y
## 3）假如安装过旧版本，先卸载旧版本
sudo yum remove docker docker-common docker-selinux docker-engine
## 4）安装 yum-util 工具包和 devicemapper 驱动依赖
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
## 5）设置 yum 源（加速 yum 下载速度）
sudo yum-config-manager --add-repo 
https://download.docker.com/linux/centos/docker-ce.repo
## 如果连接超时，可以使用 alibaba 的镜像源：
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
## 6）查看所有仓库中所有 docker 版本，并选择特定版本安装，一般可直接安装最新版
sudo yum list docker-ce --showduplicates | sort -r
## 7）安装 docker
## （1）安装最新稳定版本的方式：
sudo yum install docker-ce -y #安装的是最新稳定版本，因为 repo 中默认只
## 开启 stable 仓库
## （2）安装指定版本的方式：
sudo yum install <FQPN> -y
# 例如：
sudo yum install docker-ce-20.10.11.ce -y
## 8）启动并加入开机启动
sudo systemctl start docker #启动 docker
sudo systemctl enable docker #加入开机自启动
## 9）查看 Version，验证是否安装成功
docker version
## 若出现 Client 和 Server 两部分内容，则证明安装成功。
~~~

### 2.2、使用 Docker 开发镜像编译（doris从1.1.0版本开始提供解压即用的二进制包）

~~~shell
## 1）下载源码并解压
## 通过 wget 下载（或者手动上传下载好的压缩包）。
wget https://downloads.apache.org/doris/2.0/2.0.2/apache-doris-2.0.2-src.tar.gz
## 解压到/opt/software/ doris-2.0.2版本无法直接使用下载的源码编译，建议使用git clone
tar -zxvf apache-doris-0.15.0-incubating-src.tar.gz -C /opt/software
## 2）下载 Docker 镜像
sudo docker pull apache/doris:build-env-for-2.0
## 可以通过以下命令查看镜像是否下载完成。
sdocker images
## 3）挂载本地目录运行镜像
## 以挂载本地 Doris 源码目录的方式运行镜像，这样编译的产出二进制文件会存储在宿主机中，不会因为镜像退出而消失。同时将镜像中 maven 的 .m2 目录挂载到宿主机目录，以防止每次启动镜像编译时，重复下载 maven 的依赖库。
docker run -it -v /opt/software/.m2:/root/.m2 -v /opt/software/doris-2.0.2-src:/root/doris-2.0.2-src apache/doris:build-env-for-2.0
## 使用git拉取doris源码到目录中
cd /root/doris-2.0.2-src
git clone 
## 4）切换到 JDK 8
alternatives --set java java-1.8.0-openjdk.x86_64
alternatives --set javac java-1.8.0-openjdk.x86_64
export JAVA_HOME=/usr/lib/jvm/java-1.8.0
## 5）准备 Maven 依赖
## 编译过程会下载很多依赖，可以将我们准备好的 doris-repo.tar.gz 解压到 Docker 挂载的对应目录，来避免下载依赖的过程，加速编译。
## doris-2.0.2版本编译时还是会自动下载
tar -zxvf doris-repo.tar.gz -C /opt/software
## 也可以通过指定阿里云镜像仓库来加速下载：doris-2.0.2版本修改之后不生效
vim /opt/software/apache-doris-0.15.0-incubating-src/fe/pom.xml
## 在<repositories>标签下添加：
<repository>
 <id>aliyun</id>
 <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
</repository>
vim /opt/software/apache-doris-0.15.0-incubating-src/be/pom.xml
## 在<repositories>标签下添加：
<repository>
 <id>aliyun</id>
 <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
</repository>
## 6）编译 Doris
sh build.sh
## 如果是第一次使用 build-env-for-0.15.0 或之后的版本，第一次编译的时候要使用如下命令：
sh build.sh --clean --be --fe
## 因为 build-env-for-0.15.0 版本镜像升级了 thrift(0.9 -> 0.13)，需要通过--clean 命令强制使用新版本的 thrift 生成代码文件，否则会出现不兼容的代码。
~~~

### 2.3、安装要求

#### 2.3.1、软硬件需求

**1）Linux 操作系统要求**

| Linux 系统 | 版本         |
| ---------- | ------------ |
| CentOS     | 7.1 及以上   |
| Ubuntu     | 16.04 及以上 |

**2）软件需求**

| 软件 | 版本         |
| ---- | ------------ |
| Java | 1.8          |
| GCC  | 4.8.2 及以上 |

**3）开发测试环境**

| 模块     | CPU  | 内存  | 磁盘                 | 网络     | 实例数量 |
| -------- | ---- | ----- | -------------------- | -------- | -------- |
| Frontend | 8核+ | 8GB+  | SSD 或 SATA，10GB+ * | 千兆网卡 | 1        |
| Backend  | 8核+ | 16GB+ | SSD 或 SATA，50GB+ * | 千兆网卡 | 1-3 *    |

**4）生产环境**

| 模块     | CPU   | 内存  | 磁盘                     | 网络     | 实例数量（最低要求） |
| -------- | ----- | ----- | ------------------------ | -------- | -------------------- |
| Frontend | 16核+ | 64GB+ | SSD 或 RAID 卡，100GB+ * | 万兆网卡 | 1-3 *                |
| Backend  | 16核+ | 64GB+ | SSD 或 SATA，100G+ *     | 万兆网卡 | 3 *                  |

**5）操作系统安装要求**

~~~shell
## 1、设置系统最大打开文件句柄数和线程句柄数，记得重启虚拟机
sudo vim /etc/security/limits.conf
* soft nofile 65536
* hard nofile 65536
* soft nproc 65536
* hard nproc 65536

## 时钟同步
## Doris 的元数据要求时间精度要小于5000ms，所以所有集群所有机器要进行时钟同步，避免因为时钟问题引发的元数据不一致导致服务出现异常。

## 关闭交换分区（swap）
## Linux交换分区会给Doris带来很严重的性能问题，需要在安装之前禁用交换分区
sudo vim /etc/fstab
# 将含有/swap的行注释掉然后重启机器

## Linux文件系统
## ext4和xfs文件系统均支持。
~~~

**6）注意事项**

~~~
注1：
	1、FE 的磁盘空间主要用于存储元数据，包括日志和 image。通常从几百 MB 到几个 GB 不等。
	2、BE 的磁盘空间主要用于存放用户数据，总磁盘空间按用户总数据量 * 3（3副本）计算，然后再预留额外 40% 的空间用作后台 compaction 以及一些中间数据的存放。
	3、一台机器上虽然可以部署多个 BE，但只建议部署一个实例，同时只能部署一个 FE。如果需要 3 副本数据，那么至少需要 3 台机器各部署一个 BE 实例（而不是1台机器部署3个BE实例）。多个FE所在服务器的时钟必须保持一致（允许最多5秒的时钟偏差）
	4、测试环境也可以仅适用一个 BE 进行测试。实际生产环境，BE 实例数量直接决定了整体查询延迟。
	5、所有部署节点关闭 Swap。

注2：FE 节点的数量
	1、FE 角色分为 Follower 和 Observer，（Leader 为 Follower 组中选举出来的一种角色，以下统称 Follower）。
	2、FE 节点数据至少为1（1 个 Follower）。当部署 1 个 Follower 和 1 个 Observer 时，可以实现读高可用。当部署 3 个 Follower 时，可以实现读写高可用（HA）。
	3、Follower 的数量必须为奇数，Observer 数量随意。
	4、根据以往经验，当集群可用性要求很高时（比如提供在线业务），可以部署 3 个 Follower 和 1-3 个 Observer。如果是离线业务，建议部署 1 个 Follower 和 1-3 个 Observer。
	5、Broker 是用于访问外部数据源（如 hdfs）的进程。通常，在每台机器上部署一个 broker 实例即可。
	
注3：
	1、通常我们建议 10 ~ 100 台左右的机器，来充分发挥 Doris 的性能（其中 3 台部署 FE（HA），剩余的部署 BE）
	2、当然，Doris的性能与节点数量及配置正相关。在最少4台机器（一台 FE，三台 BE，其中一台 BE 混部一个 Observer FE 提供元数据备份），以及较低配置的情况下，依然可以平稳的运行 Doris。
	3、如果 FE 和 BE 混部，需注意资源竞争问题，并保证元数据目录和数据目录分属不同磁盘。
~~~

#### 2.3.2、默认端口

| 实例名称 | 端口名称               | 默认端口 | 通讯方向                     | 说明                                                 |
| -------- | ---------------------- | -------- | ---------------------------- | ---------------------------------------------------- |
| BE       | be_port                | 9060     | FE --> BE                    | BE 上 thrift server 的端口，用于接收来自 FE 的请求   |
| BE       | webserver_port         | 8040     | BE <--> BE                   | BE 上的 http server 的端口                           |
| BE       | heartbeat_service_port | 9050     | FE --> BE                    | BE 上心跳服务端口（thrift），用于接收来自 FE 的心跳  |
| BE       | brpc_port              | 8060     | FE <--> BE, BE <--> BE       | BE 上的 brpc 端口，用于 BE 之间通讯                  |
| FE       | http_port              | 8030     | FE <--> FE，用户 <--> FE     | FE 上的 http server 端口                             |
| FE       | rpc_port               | 9020     | BE --> FE, FE <--> FE        | FE 上的 thrift server 端口，每个fe的配置需要保持一致 |
| FE       | query_port             | 9030     | 用户 <--> FE                 | FE 上的 mysql server 端口                            |
| FE       | arrow_flight_sql_port  | 9040     | 用户 <--> FE                 | FE 上的 Arrow Flight SQL server 端口                 |
| FE       | edit_log_port          | 9010     | FE <--> FE                   | FE 上的 bdbje 之间通信用的端口                       |
| Broker   | broker_ipc_port        | 8000     | FE --> Broker, BE --> Broker | Broker 上的 thrift server，用于接收请求              |

注：

1. 当部署多个 FE 实例时，要保证 FE 的 http_port 配置相同。
2. 部署前请确保各个端口在应有方向上的访问权限。

### 2.4、集群部署

部署规划：

| hadoop102    | hadoop103      | hadoop104      |
| ------------ | -------------- | -------------- |
| FE（Leader） | FE（Follower） | FE（Observer） |
| BE           | BE             | BE             |
| Broker       | Broker         | Broker         |

注：生产环境建议 FE 和 BE 分开。

#### 2.4.1、创建目录并拷贝编译后的文件

~~~
mkdir /opt/module/doris-2.0.2
cp -r /opt/software/doris-2.0.2-src/doris/output/* /opt/module/doris-2.0.2
cp -r /opt/software/doris-2.0.2-src/doris/fs_brokers/apache_hdfs_broker/output/* /opt/module/doris-2.0.2
~~~

#### 2.4.2、部署 FE 节点

**1）创建 fe 元数据存储的目录**

~~~
mkdir /opt/module/doris-2.0.2/doris-meta
~~~

**2）修改 fe 的配置文件**

~~~
vim /opt/module/doris-2.0.2/fe/conf/fe.conf
#配置文件中指定元数据路径：
meta_dir = /opt/module/doris-2.0.2/doris-meta
#修改绑定 ip（每台机器修改成自己的 ip）
priority_networks = 192.168.8.101/24
~~~

**注意：**

~~~
	1、生产环境强烈建议单独指定目录不要放在 Doris 安装目录下，最好是单独的磁盘（如果有SSD最好）。
	2、如果机器有多个 ip, 比如内网外网, 虚拟机 docker 等, 需要进行 ip 绑定，才能正确识 别。
	3、JAVA_OPTS 默认 java 最大堆内存为 4GB，建议生产环境调整至 8G 以上。
~~~

**启动命令：**

~~~shell
/opt/module/doris-2.0.2/fe/bin/start_fe.sh --daemon
~~~

#### 2.4.3、配置 BE 节点

**1）分发 BE**

~~~
xsync /opt/module/doris-2.0.2/be
~~~

**2）创建 BE 数据存放目录（每个节点）**

~~~
mkdir /opt/module/doris-2.0.2/storage01 /opt/module/doris-2.0.2/storage02
~~~

**3）修改 BE 的配置文件（每个节点）**

~~~
vim /opt/module/doris-2.0.2/be/conf/be.conf
#配置文件中指定数据存放路径：
storage_root_path = /opt/module/doris-2.0.2/storage01;/opt/module/doris-2.0.2/storage02
#修改绑定 ip（每台机器修改成自己的 ip）
priority_networks = 192.168.52.102/24
~~~

**注意：**

~~~
	1、storage_root_path 默认在 be/storage 下，需要手动创建该目录。多个路径之间使用英文状态的分号;分隔（最后一个目录后不要加）。
	2、可以通过路径区别存储目录的介质，HDD 或 SSD。可以添加容量限制在每个路径的末尾，通过英文状态逗号，隔开，如：storage_root_path=/home/disk1/doris.HDD,50;/home/disk2/doris.SSD,
10;/home/disk2/doris
	说明：
        /home/disk1/doris.HDD,50，表示存储限制为 50GB，HDD;
        /home/disk2/doris.SSD,10，存储限制为 10GB，SSD；
        /home/disk2/doris，存储限制为磁盘最大容量，默认为 HDD
	3、如果机器有多个 IP, 比如内网外网, 虚拟机 docker 等, 需要进行 IP 绑定，才能正确识别。
	4、如果 be 部署在 hadoop 集群中，注意调整 be.conf 中的 webserver_port = 8040 ,以免造成端口冲突
~~~

#### 2.4.4、在 FE 中添加所有 BE 节点

BE 节点需要先在 FE 中添加，才可加入集群。可以使用 mysql-client 连接到 FE。

**1）安装 MySQL Client**

~~~shell
# （1）创建目录
mkdir /opt/software/mysql-client/
## （2）上传相关以下三个 rpm 包到/opt/software/mysql-client/
## mysql-community-client-5.7.28-1.el7.x86_64.rpm
## mysql-community-common-5.7.28-1.el7.x86_64.rpm
## mysql-community-libs-5.7.28-1.el7.x86_64.rpm
## （3）检查当前系统是否安装过 MySQL
sudo rpm -qa|grep mariadb
#如果存在，先卸载
sudo rpm -e --nodeps mariadb mariadb-libs mariadb-server
## （4）安装
rpm -ivh /opt/software/mysql-client/*
~~~

**2）使用 MySQL Client 连接 FE**

~~~shell
mysql -h hadoop102 -P 9030 -uroot
## 默认 root 无密码，通过以下命令修改 root 密码。
SET PASSWORD FOR 'root' = PASSWORD('123456');
~~~

**3）添加 BE**

~~~shell
ALTER SYSTEM ADD BACKEND "hadoop102:9050";
ALTER SYSTEM ADD BACKEND "hadoop103:9050";
ALTER SYSTEM ADD BACKEND "hadoop104:9050";
~~~

**4）查看 BE 状态**

~~~sql
SHOW PROC '/backends';
~~~

#### 2.4.5、启动 BE

**1）启动 BE（每个节点）**

~~~sh
/opt/module/doris-2.0.2/be/bin/start_be.sh --daemon
~~~

**2）查看 BE 状态**

~~~shell
## 1、使用mysql连接FE
mysql -u root -p -P 9030 -h hadoop102
## 2、执行命令
SHOW PROC '/backends';
## Alive 为 true 表示该 BE 节点存活。
~~~

#### 2.4.6、部署 FS_Broker（可选）

~~~shell
	Broker 以插件的形式，独立于 Doris 部署。如果需要从第三方存储系统导入数据，需要部署相应的 Broker，默认提供了读取 HDFS、百度云 BOS 及 Amazon S3 的 fs_broker。
	fs_broker 是无状态的，建议每一个 FE 和 BE 节点都部署一个 Broker。
	1）编译 FS_BROKER 并拷贝文件
    （1）进入源码目录下的 fs_brokers 目录，使用 sh build.sh 进行编译
    （2）拷贝源码 fs_broker 的 output 目录下的相应 Broker 目录到需要部署的所有节点上，改名为: apache_hdfs_broker。建议和 BE 或者 FE 目录保持同级。方法同 2.2。
	2）启动 Broker(每个节点)
	/opt/module/doris-2.0.2/apache_hdfs_broker/bin/start_broker.sh --daemon
	3）添加 Broker
	要让 Doris 的 FE 和 BE 知道 Broker 在哪些节点上，通过 sql 命令添加 Broker 节点列表。
	（1）使用 mysql-client 连接启动的 FE，执行以下命令：
	mysql -h hadoop102 -P 9030 -uroot -p
	ALTER SYSTEM ADD BROKER hdfs_broker "hadoop102:8000","hadoop103:8000","hadoop104:8000";
	其中 broker_host 为 Broker 所在节点 ip；broker_ipc_port 在 Broker 配置文件中的conf/apache_hdfs_broker.conf。
	4）查看 Broker 状态
	使用 mysql-client 连接任一已启动的 FE，执行以下命令查看 Broker 状态：
	SHOW PROC "/brokers";
	
注：在生产环境中，所有实例都应使用守护进程启动，以保证进程退出后，会被自动拉起，如 Supervisor（opens new window）。如需使用守护进程启动，在 0.9.0 及之前版本中，需要修改各个 start_xx.sh 脚本，去掉最后的 & 符号。从 0.10.0 版本开始，直接调用 sh 
start_xx.sh 启动即可。
~~~

### 2.5、扩容和缩容

Doris 可以很方便的扩容和缩容 FE、BE、Broker 实例。

#### 2.5.1、FE 扩容和缩容

可以通过将 FE 扩容至 3 个以上节点来实现 FE 的高可用。

**1）使用 MySQL 登录客户端后，可以使用 sql 命令查看 FE 状态，目前就一台 FE**

~~~sql
mysql -u root -p -P 9030 -h hadoop102
show frontends\G

-- 也可以通过页面访问进行监控，访问 fe_host:8030，账户为 root，密码默认为空不用填写。
~~~

**2）增加 FE 节点**

~~~sql
	FE 分为 Follower 和 Observer 两种角色，其中 Follower 角色会选举出一个 Follower 节点作为Master。 默认一个集群，只能有一个 Master 状态的 Follower 角色，可以有多个 Follower 和 Observer同时需保证 Follower 角色为奇数个。其中所有 Follower 角色组成一个选举组，如果 Master 状态的 Follower宕机，则剩下的 Follower 会自动选出新的 Master，保证写入高可用。Observer 同步 Master 的数据，但是参加选举。如果只部署一个 FE，则 FE 默认就是 Master。
	第一个启动的 FE 自动成为 Master。在此基础上，可以添加若干 Follower 和 Observer。
	ALTER SYSTEM ADD FOLLOWER "hadoop103:9010";
	ALTER SYSTEM ADD OBSERVER "hadoop104:9010";
~~~

**3）配置及启动 Follower 和 Observer**

第一次启动时，启动命令需要添加参--helper leader 主机: edit_log_port

~~~shell
start_fe.sh --helper hadoop102:9010 --daemon
~~~

**4）查看运行状态**

~~~sql
-- 使用 mysql-client 连接到任一已启动的 FE。
SHOW PROC '/frontends';
~~~

**5）删除 FE 节点命令**

~~~sql
ALTER SYSTEM DROP FOLLOWER[OBSERVER] "fe_host:edit_log_port";
-- 注意：删除 Follower FE 时，确保最终剩余的 Follower（包括 Leader）节点为奇数。
-- 如果需要对删除之后的节点再次上线，需要清空该节点的fe元数据目录
~~~

#### 2.5.2、BE 扩容和缩容

**1）增加 BE 节点**

~~~
在 MySQL 客户端，通过 ALTER SYSTEM ADD BACKEND 命令增加 BE 节点。
~~~

**2）DROP 方式删除 BE 节点（不推荐）**

~~~
注意：DROP BACKEND 会直接删除该 BE，并且其上的数据将不能再恢复！！！所以我们强烈不推荐使用 DROP BACKEND 这种方式删除 BE 节点。当你使用这个语句时，会有对应的防误操作提示。
~~~

**3）DECOMMISSION 方式删除 BE 节点（推荐）**

~~~shell
alter system decommission backend "be_host:9050";
## 1、该命令用于安全删除 BE 节点。命令下发后，Doris 会尝试将该 BE 上的数据向其他 BE 节点迁移，当所有数据都迁移完成后，Doris 会自动删除该节点。
## 2、该命令是一个异步操作。执行后，可以通过 SHOW PROC '/backends'; 看到该 BE 节点的 isDecommission 状态为 true。表示该节点正在进行下线。
## 3、该命令不一定执行成功。比如剩余 BE 存储空间不足以容纳下线 BE 上的数据，或者剩余机器数量不满足最小副本数时，该命令都无法完成，并且 BE 会一直处于isDecommission 为 true 的状态。
## 4、DECOMMISSION 的进度，可以通过 SHOW PROC '/backends'; 中的 TabletNum 查看，如果正在进行，TabletNum 将不断减少。
## 5、该操作可以通过如下命令取消：
CANCEL DECOMMISSION BACKEND "be_host:be_heartbeat_service_port";
## 取消后，该 BE 上的数据将维持当前剩余的数据量。后续 Doris 重新进行负载均衡。
~~~

#### 2.5.3、Broker 扩容缩容

~~~shell
## Broker 实例的数量没有硬性要求。通常每台物理机部署一个即可。Broker 的添加和删除可以通过以下命令完成：
alter system add broker broker_name "broker_host:broker_ipc_port";
alter system drop broker broker_name "broker_host:broker_ipc_port";
alter system drop all broker broker_name;
## 注：Broker 是无状态的进程，可以随意启停。当然，停止后，正在其上运行的作业会失败，重试即可。
~~~

## 三、数据表设计

### 3.1、创建用户和数据库

**1）创建 test 用户**

~~~sql
mysql -h hadoop102 -P 9030 -uroot -p
create user 'test' identified by 'test';
~~~

**2）创建数据库**

~~~sql
create database example_db;
~~~

**3）用户授权**

~~~sql
grant all on example_db to test;
~~~

### 3.2、基本概念

在 Doris 中，数据都以关系表（Table）的形式进行逻辑上的描述。

#### 3.2.1、Row & Column

~~~
一张表包括行（Row）和列（Column）。Row 即用户的一行数据。Column 用于描述一行数据中不同的字段。
	1、在默认的数据模型中，Column 只分为排序列和非排序列。存储引擎会按照排序列对数据进行排序存储，并建立稀疏索引，以便在排序数据上进行快速查找。
	2、而在聚合模型中，Column 可以分为两大类：Key 和 Value。从业务角度看，Key 和Value 可以分别对应维度列和指标列。从聚合模型的角度来说，Key 列相同的行，会聚合成一行。其中 Value 列的聚合方式由用户在建表时指定。
~~~

#### 3.2.2、Partition & Tablet

~~~
在 Doris 的存储引擎中，用户数据首先被划分成若干个分区（Partition），划分的规则通常是按照用户指定的分区列进行范围划分，比如按时间划分。而在每个分区内，数据被进一步的按照 Hash 的方式分桶，分桶的规则是要找用户指定的分桶列的值进行 Hash 后分桶。每个分桶就是一个数据分片（Tablet），也是数据划分的最小逻辑单元。
	1、Tablet 之间的数据是没有交集的，独立存储的。Tablet 也是数据移动、复制等操作
的最小物理存储单元。
	2、Partition 可以视为是逻辑上最小的管理单元。数据的导入与删除，都可以或仅能针对一个 Partition 进行。
~~~

### 3.3、建表示例

#### 3.3.1、建表语法

使用 CREATE TABLE 命令建立一个表(Table)。更多详细参数可以查看：

~~~sql
HELP CREATE TABLE;
~~~

建表语法：

~~~sql
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] [database.]table_name
 (column_definition1[, column_definition2, ...]
 [, index_definition1[, index_definition12,]])
 [ENGINE = [olap|mysql|broker|hive]]
 [key_desc]
 [COMMENT "table comment"];
 [partition_desc]
 [distribution_desc]
 [rollup_index]
 [PROPERTIES ("key"="value", ...)]
 [BROKER PROPERTIES ("key"="value", ...)];
 
-- Doris 的建表是一个同步命令，命令返回成功，即表示建表成功。 
-- Doris 支持支持单分区和复合分区两种建表方式。
~~~

1）复合分区：既有分区也有分桶

~~~
	第一级称为 Partition，即分区。用户可以指定某一维度列作为分区列（当前只支持整型和时间类型的列），并指定每个分区的取值范围。
	第二级称为 Distribution，即分桶。用户可以指定一个或多个维度列以及桶数对数据进行 HASH 分布。
~~~

2）单分区：只做 HASH 分布，即只分桶。

#### 3.3.2、字段类型

|            类型             |      长度      |                             范围                             |
| :-------------------------: | :------------: | :----------------------------------------------------------: |
|           TINYINT           |     1 字节     |                   范围：-2^7 + 1 ~ 2^7 - 1                   |
|          SMALLINT           |     2 字节     |                  范围：-2^15 + 1 ~ 2^15 - 1                  |
|             INT             |     4 字节     |                  范围：-2^31 + 1 ~ 2^31 - 1                  |
|           BIGINT            |     8 字节     |                  范围：-2^63 + 1 ~ 2^63 - 1                  |
|          LARGEINT           |    16 字节     |                 范围：-2^127 + 1 ~ 2^127 - 1                 |
|            FLOAT            |     4 字节     |                        支持科学计数法                        |
|           DOUBLE            |    12 字节     |                        支持科学计数法                        |
| DECIMAL[(precision, scale)] |    16 字节     | 保证精度的小数类型。默认是 DECIMAL(10, 0) ；precision: 1 ~ 27 ；scale: 0 ~ 9 其中整数部分为 1 ~ 18 不支持科学计数法 |
|            DATE             |     3 字节     |                范围：0000-01-01 ~ 9999-12-31                 |
|          DATETIME           |     8 字节     |       范围：0000-01-01 00:00:00 ~ 9999-12- 31 23:59:59       |
|       CHAR[(length)]        |                |           定长字符串。长度范围：1 ~ 255。默 认为 1           |
|      VARCHAR[(length)]      |                |               变长字符串。长度范围：1 ~ 65533                |
|           BOOLEAN           |                |         与 TINYINT 一样，0 代表 false，1 代 表 true          |
|             HLL             | 1~16385 个字节 | hll 列类型，不需要指定长度和默认值、 长度根据数据的聚合 程度系统内控制，并且 HLL 列只能通 过 配 套 的 hll_union_agg 、 Hll_cardinality、hll_hash 进行查询或使 用 |
|           BITMAP            |                | bitmap 列类型，不需要指定长度和默 认值。表示整型的集合，元素最大支持 到 2^64 - 1 |
|           STRING            |                | 变长字符串，0.15 版本支持，最大支 持 2147483643 字节（2GB-4），长度 还受 be 配置`string_type_soft_limit`,  实际能存储的最大长度取两者最小 值。只能用在 value 列，不能用在 key  列和分区、分桶列 |

~~~
注：聚合模型在定义字段类型后，可以指定字段的 agg_type 聚合类型，如果不指定，则该列为 key 列。否则，该列为 value 列, 类型包括：SUM、MAX、MIN、REPLACE。
~~~

#### 3.3.2、建表示例

**我们以一个建表操作来说明 Doris 的数据划分。**

##### 3.3.2.1、Range Partition

~~~sql
CREATE TABLE IF NOT EXISTS example_db.example_range_tbl
(
    `user_id` LARGEINT NOT NULL COMMENT "用户id",
    `date` DATE NOT NULL COMMENT "数据灌入日期时间",
    `timestamp` DATETIME NOT NULL COMMENT "数据灌入的时间戳",
    `city` VARCHAR(20) COMMENT "用户所在城市",
    `age` SMALLINT COMMENT "用户年龄",
    `sex` TINYINT COMMENT "用户性别",
    `last_visit_date` DATETIME REPLACE DEFAULT "1970-01-01 00:00:00" COMMENT "用户最后一次访问时间",
    `cost` BIGINT SUM DEFAULT "0" COMMENT "用户总消费",
    `max_dwell_time` INT MAX DEFAULT "0" COMMENT "用户最大停留时间",
    `min_dwell_time` INT MIN DEFAULT "99999" COMMENT "用户最小停留时间"
)
ENGINE=OLAP
AGGREGATE KEY(`user_id`, `date`, `timestamp`, `city`, `age`, `sex`)
PARTITION BY RANGE(`date`)
(
    PARTITION `p201701` VALUES LESS THAN ("2017-02-01"),
    PARTITION `p201702` VALUES LESS THAN ("2017-03-01"),
    PARTITION `p201703` VALUES LESS THAN ("2017-04-01")
)
DISTRIBUTED BY HASH(`user_id`) BUCKETS 16
PROPERTIES
(
    "replication_num" = "3",
    "storage_medium" = "SSD",
    "storage_cooldown_time" = "2018-01-01 12:00:00"
);
~~~

##### 3.3.2.2、List Partition

~~~sql
CREATE TABLE IF NOT EXISTS example_db.example_list_tbl
(
    `user_id` LARGEINT NOT NULL COMMENT "用户id",
    `date` DATE NOT NULL COMMENT "数据灌入日期时间",
    `timestamp` DATETIME NOT NULL COMMENT "数据灌入的时间戳",
    `city` VARCHAR(20) NOT NULL COMMENT "用户所在城市",
    `age` SMALLINT COMMENT "用户年龄",
    `sex` TINYINT COMMENT "用户性别",
    `last_visit_date` DATETIME REPLACE DEFAULT "1970-01-01 00:00:00" COMMENT "用户最后一次访问时间",
    `cost` BIGINT SUM DEFAULT "0" COMMENT "用户总消费",
    `max_dwell_time` INT MAX DEFAULT "0" COMMENT "用户最大停留时间",
    `min_dwell_time` INT MIN DEFAULT "99999" COMMENT "用户最小停留时间"
)
ENGINE=olap
AGGREGATE KEY(`user_id`, `date`, `timestamp`, `city`, `age`, `sex`)
PARTITION BY LIST(`city`)
(
    PARTITION `p_cn` VALUES IN ("Beijing", "Shanghai", "Hong Kong"),
    PARTITION `p_usa` VALUES IN ("New York", "San Francisco"),
    PARTITION `p_jp` VALUES IN ("Tokyo")
)
DISTRIBUTED BY HASH(`user_id`) BUCKETS 16
PROPERTIES
(
    "replication_num" = "3",
    "storage_medium" = "SSD",
    "storage_cooldown_time" = "2018-01-01 12:00:00"
);
~~~

### 3.4、数据划分

以 3.3.2 的建表示例来理解。

#### 3.4.1、列定义

~~~
这里我们只以 AGGREGATE KEY 数据模型为例进行说明。更多数据模型参阅 Doris 数据模型。
列的基本类型，可以通过在 mysql-client 中执行 HELP CREATE TABLE; 查看。
	AGGREGATE KEY 数据模型中，所有没有指定聚合方式（SUM、REPLACE、MAX、MIN）的列视为 Key 列。而其余则为 Value 列。
定义列时，可参照如下建议：
	1、Key 列必须在所有 Value 列之前。
	2、尽量选择整型类型。因为整型类型的计算和查找效率远高于字符串。
	3、对于不同长度的整型类型的选择原则，遵循 够用即可。
	4、对于 VARCHAR 和 STRING 类型的长度，遵循 够用即可。
~~~

#### 3.4.2、分区与分桶

~~~
	Doris 支持两层的数据划分。第一层是 Partition，支持 Range 和 List 的划分方式。第二层是 Bucket（Tablet），支持 Hash 和 Random 的划分方式。
	也可以仅使用一层分区，建表时如果不写分区的语句即可，此时Doris会生成一个默认的分区，对用户是透明的。使用一层分区时，只支持 Bucket 划分。
~~~

##### 3.4.2.1、Partition

~~~
Partition 列可以指定一列或多列。分区类必须为 KEY 列。多列分区的使用方式在后面介绍。
    ➢ 不论分区列是什么类型，在写分区值时，都需要加双引号。
    ➢ 分区数量理论上没有上限。
    ➢ 当不使用 Partition 建表时，系统会自动生成一个和表名同名的，全值范围的Partition。该 Partition 对用户不可见，并且不可删改。
~~~

**1） Range 分区**

​		1、分区列通常为时间列，以方便的管理新旧数据。
​				2、Range 分区支持的列类型：[<font color="red">DATE,DATETIME,TINYINT,SMALLINT,INT,BIGINT,LARGEINT</font>]
​				3、Partition 支持通过 <font color="red">VALUES LESS THAN (...) </font>仅指定上界，系统会将前一个分区的上界作为该分区的		下界，生成一个左闭右开的区间。也支持通过 <font color="red"> VALUES [...) </font>指定上下界，生成一个左闭右开的区间。
​				4、同时，也支持通过<font color="red"> `FROM(...) TO (...) INTERVAL ...` </font>来批量创建分区。
​				5、分区的删除不会改变已存在分区的范围。删除分区可能出现空洞。通过<font color="red"> VALUES LESS THAN </font>语句增		加分区时，分区的下界紧接上一个分区的上界。

~~~
如上 example_range_tbl 示例，当建表完成后，会自动生成如下3个分区：

p201701: [MIN_VALUE,  2017-02-01)
p201702: [2017-02-01, 2017-03-01)
p201703: [2017-03-01, 2017-04-01)

当我们增加一个分区 p201705 VALUES LESS THAN ("2017-06-01")，分区结果如下：

p201701: [MIN_VALUE,  2017-02-01)
p201702: [2017-02-01, 2017-03-01)
p201703: [2017-03-01, 2017-04-01)
p201705: [2017-04-01, 2017-06-01)

此时我们删除分区 p201703，则分区结果如下：

p201701: [MIN_VALUE,  2017-02-01)
p201702: [2017-02-01, 2017-03-01)
p201705: [2017-04-01, 2017-06-01)

注意到 p201702 和 p201705 的分区范围并没有发生变化，而这两个分区之间，出现了一个空洞：[2017-03-01, 2017-04-01)。即如果导入的数据范围在这个空洞范围内，是无法导入的。

继续删除分区 p201702，分区结果如下：

p201701: [MIN_VALUE,  2017-02-01)
p201705: [2017-04-01, 2017-06-01)

空洞范围变为：[2017-02-01, 2017-04-01)

现在增加一个分区 p201702new VALUES LESS THAN ("2017-03-01")，分区结果如下：

p201701:    [MIN_VALUE,  2017-02-01)
p201702new: [2017-02-01, 2017-03-01)
p201705:    [2017-04-01, 2017-06-01)

可以看到空洞范围缩小为：[2017-03-01, 2017-04-01)

现在删除分区 p201701，并添加分区 p201612 VALUES LESS THAN ("2017-01-01")，分区结果如下：

p201612:    [MIN_VALUE,  2017-01-01)
p201702new: [2017-02-01, 2017-03-01)
p201705:    [2017-04-01, 2017-06-01) 

即出现了一个新的空洞：[2017-01-01, 2017-02-01)
~~~

**Range分区除了上述我们看到的单列分区，也支持多列分区，示例如下：**

~~~
PARTITION BY RANGE(`date`, `id`)
(
    PARTITION `p201701_1000` VALUES LESS THAN ("2017-02-01", "1000"),
    PARTITION `p201702_2000` VALUES LESS THAN ("2017-03-01", "2000"),
    PARTITION `p201703_all`  VALUES LESS THAN ("2017-04-01")
)

在以上示例中，我们指定 date(DATE 类型) 和 id(INT 类型) 作为分区列。以上示例最终得到的分区如下：

    * p201701_1000:    [(MIN_VALUE,  MIN_VALUE), ("2017-02-01", "1000")   )
    * p201702_2000:    [("2017-02-01", "1000"),  ("2017-03-01", "2000")   )
    * p201703_all:     [("2017-03-01", "2000"),  ("2017-04-01", MIN_VALUE)) 

注意，最后一个分区用户缺省只指定了 date 列的分区值，所以 id 列的分区值会默认填充 MIN_VALUE。当用户插入数据时，分区列值会按照顺序依次比较，最终得到对应的分区。举例如下：

    * 数据  -->  分区
    * 2017-01-01, 200     --> p201701_1000
    * 2017-01-01, 2000    --> p201701_1000
    * 2017-02-01, 100     --> p201701_1000
    * 2017-02-01, 2000    --> p201702_2000
    * 2017-02-15, 5000    --> p201702_2000
    * 2017-03-01, 2000    --> p201703_all
    * 2017-03-10, 1       --> p201703_all
    * 2017-04-01, 1000    --> 无法导入
    * 2017-05-01, 1000    --> 无法导入

Range分区同样支持批量分区， 通过语句 FROM ("2022-01-03") TO ("2022-01-06") INTERVAL 1 DAY 批量创建按天划分的分区：2022-01-03到2022-01-06（不含2022-01-06日），分区结果如下：

p20220103:    [2022-01-03,  2022-01-04)
p20220104:    [2022-01-04,  2022-01-05)
p20220105:    [2022-01-05,  2022-01-06)
~~~

**2）List 分区**

​	1、分区列支持<font color="red">`BOOLEAN, TINYINT, SMALLINT, INT, BIGINT, LARGEINT, DATE, DATETIME, CHAR, VARCHAR` </font>数据类型，分区值为枚举值。只有当数据为目标分区枚举值其中之一时，才可以命中分区。

​	2、Partition 支持通过<font color="red">`VALUES IN (...)` </font>来指定每个分区包含的枚举值。

​	3、下面通过示例说明，进行分区的增删操作时，分区的变化。

~~~
如上 example_list_tbl 示例，当建表完成后，会自动生成如下3个分区：

p_cn: ("Beijing", "Shanghai", "Hong Kong")
p_usa: ("New York", "San Francisco")
p_jp: ("Tokyo")

当我们增加一个分区 p_uk VALUES IN ("London")，分区结果如下：

p_cn: ("Beijing", "Shanghai", "Hong Kong")
p_usa: ("New York", "San Francisco")
p_jp: ("Tokyo")
p_uk: ("London")

当我们删除分区 p_jp，分区结果如下：

p_cn: ("Beijing", "Shanghai", "Hong Kong")
p_usa: ("New York", "San Francisco")
p_uk: ("London")
~~~

**List分区也支持多列分区，示例如下：**

~~~
PARTITION BY LIST(`id`, `city`)
(
    PARTITION `p1_city` VALUES IN (("1", "Beijing"), ("1", "Shanghai")),
    PARTITION `p2_city` VALUES IN (("2", "Beijing"), ("2", "Shanghai")),
    PARTITION `p3_city` VALUES IN (("3", "Beijing"), ("3", "Shanghai"))
)

在以上示例中，我们指定 id(INT 类型) 和 city(VARCHAR 类型) 作为分区列。以上示例最终得到的分区如下：

  * p1_city: [("1", "Beijing"), ("1", "Shanghai")]
  * p2_city: [("2", "Beijing"), ("2", "Shanghai")]
  * p3_city: [("3", "Beijing"), ("3", "Shanghai")]

当用户插入数据时，分区列值会按照顺序依次比较，最终得到对应的分区。举例如下：

  * 数据  --->  分区
  * 1, Beijing     ---> p1_city
  * 1, Shanghai    ---> p1_city
  * 2, Shanghai    ---> p2_city
  * 3, Beijing     ---> p3_city
  * 1, Tianjin     ---> 无法导入
  * 4, Beijing     ---> 无法导入
~~~

##### 3.4.2.2、Bucket

1、如果使用了 Partition，则 `DISTRIBUTED ...` 语句描述的是数据在**各个分区内**的划分规则。如果不使用 Partition，则描述的是对整个表的数据的划分规则。

2、分桶列可以是多列，Aggregate 和 Unique 模型<font color="red">必须为 Key 列</font>，Duplicate 模型可以是 key 列和 value 列。分桶列可以和 Partition 列相同或不同。

3、分桶列的选择，是在**查询吞吐**和**查询并发**之间的一种权衡：

​		1）如果选择多个分桶列，则数据分布更均匀。如果一个查询条件不包含所有分桶列的等值条件，那么该		查询会触发所有分桶同时扫描，这样查询的吞吐会增加，单个查询的延迟随之降低。这个方式适合大吞吐低		并发的查询场景。

​		2）如果仅选择一个或少数分桶列，则对应的点查询可以仅触发一个分桶扫描。此时，当多个点查询并发		时，这些查询有较大的概率分别触发不同的分桶扫描，各个查询之间的IO影响较小（尤其当不同桶分布在不		同磁盘上时），所以这种方式适合高并发的点查询场景。

4、AutoBucket: 根据数据量，计算分桶数。 对于分区表，可以根据历史分区的数据量、机器数、盘数，确定一个分桶。

5、分桶的数量理论上没有上限。

##### 3.4.2.3、**关于 Partition 和 Bucket 的数量和数据量的建议。**

- 一个表的 Tablet 总数量等于 (Partition num * Bucket num)。
- 一个表的 Tablet 数量，在不考虑扩容的情况下，推荐略多于整个集群的磁盘数量。
- 单个 Tablet 的数据量理论上没有上下界，但建议在 1G - 10G 的范围内。如果单个 Tablet 数据量过小，则数据的聚合效果不佳，且元数据管理压力大。如果数据量过大，则不利于副本的迁移、补齐，且会增加 Schema Change 或者 Rollup 操作失败重试的代价（这些操作失败重试的粒度是 Tablet）。
- 当 Tablet 的数据量原则和数量原则冲突时，建议优先考虑数据量原则。
- 在建表时，每个分区的 Bucket 数量统一指定。但是在动态增加分区时（`ADD PARTITION`），可以单独指定新分区的 Bucket 数量。可以利用这个功能方便的应对数据缩小或膨胀。
- 一个 Partition 的 Bucket 数量一旦指定，不可更改。所以在确定 Bucket 数量时，需要预先考虑集群扩容的情况。比如当前只有 3 台 host，每台 host 有 1 块盘。如果 Bucket 的数量只设置为 3 或更小，那么后期即使再增加机器，也不能提高并发度。
- 举一些例子：假设在有10台BE，每台BE一块磁盘的情况下。如果一个表总大小为 500MB，则可以考虑4-8个分片。5GB：8-16个分片。50GB：32个分片。500GB：建议分区，每个分区大小在 50GB 左右，每个分区16-32个分片。5TB：建议分区，每个分区大小在 50GB 左右，每个分区16-32个分片。

注：表的数据量可以通过 [`SHOW DATA`](https://doris.apache.org/zh-CN/docs/sql-manual/sql-reference/Show-Statements/SHOW-DATA) 命令查看，结果除以副本数，即表的数据量。

##### 3.4.2.4、**关于 Random Distribution 的设置以及使用场景。**

- 如果 OLAP 表没有更新类型的字段，将表的数据分桶模式设置为 RANDOM，则可以避免严重的数据倾斜(数据在导入表对应的分区的时候，单次导入作业每个 batch 的数据将随机选择一个tablet进行写入)。
- 当表的分桶模式被设置为RANDOM 时，因为没有分桶列，无法根据分桶列的值仅对几个分桶查询，对表进行查询的时候将对命中分区的全部分桶同时扫描，该设置适合对表数据整体的聚合查询分析而不适合高并发的点查询。
- 如果 OLAP 表的是 Random Distribution 的数据分布，那么在数据导入的时候可以设置单分片导入模式（将 `load_to_single_tablet` 设置为 true），那么在大数据量的导入的时候，一个任务在将数据写入对应的分区时将只写入一个分片，这样将能提高数据导入的并发度和吞吐量，减少数据导入和 Compaction 导致的写放大问题，保障集群的稳定性。

##### 3.4.2.5、复合分区与单分区

**复合分区**

~~~
第一级称为 Partition，即分区。用户可以指定某一维度列作为分区列（当前只支持整型和时间类型的列），并指定每个分区的取值范围。
第二级称为 Distribution，即分桶。用户可以指定一个或多个维度列以及桶数对数据进行 HASH 分布 或者不指定分桶列设置成 Random Distribution 对数据进行随机分布。
~~~

##### 3.4.2.6、以下场景推荐使用复合分区

- 有时间维度或类似带有有序值的维度，可以以这类维度列作为分区列。分区粒度可以根据导入频次、分区数据量等进行评估。
- 历史数据删除需求：如有删除历史数据的需求（比如仅保留最近N 天的数据）。使用复合分区，可以通过删除历史分区来达到目的。也可以通过在指定分区内发送 DELETE 语句进行数据删除。
- 解决数据倾斜问题：每个分区可以单独指定分桶数量。如按天分区，当每天的数据量差异很大时，可以通过指定分区的分桶数，合理划分不同分区的数据,分桶列建议选择区分度大的列。

**用户也可以不使用复合分区，即使用单分区。则数据只做 HASH 分布。**

#### 3.4.3、PROPERTIES

设置表属性。目前支持以下属性：

##### 3.4.3.1、replication_num

每个 Tablet 的副本数量。默认为 3，建议保持默认即可。

在建表语句中，所有 Partition  中的 Tablet 副本数量统一指定。而在增加新分区时，可以单独指定新分区中 Tablet 的副本 数量。 

副本数量可以在运行时修改。强烈建议保持奇数。 

**最大副本数量取决于集群中独立 IP 的数量（注意不是 BE 数量）**。Doris 中副本分布的 原则是，不允许同一个 Tablet 的副本分布在同一台物理机上，而识别物理机即通过 IP。所 以，即使在同一台物理机上部署了 3 个或更多 BE 实例，如果这些 BE 的 IP 相同，则依然只 能设置副本数为 1。

 对于一些小，并且更新不频繁的维度表，可以考虑设置更多的副本数。这样在 Join 查询 时，可以有更大的概率进行本地数据 Join。

##### 3.4.3.2、replication_allocation

根据 Tag 设置副本分布情况。该属性可以完全覆盖 `replication_num` 属性的功能。

##### 3.4.3.3、min_load_replica_num

设定数据导入成功所需的最小副本数，默认值为-1。当该属性小于等于0时，表示导入数据仍需多数派副本成功。

##### 3.4.3.4、is_being_synced

用于标识此表是否是被CCR复制而来并且正在被syncer同步，默认为 `false`。

如果设置为 `true`：
		`colocate_with`，`storage_policy`属性将被擦除
`dynamic partition`，`auto bucket`功能将会失效，即在`show create table`中显示开启状态，但不会实际生效。当`is_being_synced`被设置为 `false` 时，这些功能将会恢复生效。

这个属性仅供CCR外围模块使用，在CCR同步的过程中不要手动设置。

##### 3.4.3.5、storage_medium & storage_cooldown_time

数据存储介质。`storage_medium` 用于声明表数据的初始存储介质，而 `storage_cooldown_time` 用于设定到期时间。示例：

```text
"storage_medium" = "SSD",
"storage_cooldown_time" = "2020-11-20 00:00:00"
```

这个示例表示数据存放在 SSD 中，并且在 2020-11-20 00:00:00 到期后，会自动迁移到 HDD 存储上。

~~~
	BE 的数据存储目录可以显式的指定为 SSD 或者 HDD（通过 .SSD 或者 .HDD 后缀区分）。建表时，可以统一指定所有 Partition 初始存储的介质。注意，后缀作用是显式指定磁盘介质，而不会检查是否与实际介质类型相符。
	默认初始存储介质可通过 fe 的配置文件 fe.conf 中指定 default_storage_medium=xxx，如果没有指定，则默认为 HDD。如果指定为 SSD，则数据初始存放在 SSD 上。
	如果没有指定 storage_cooldown_time，则默认 30 天后，数据会从 SSD 自动迁移到 HDD 上。如果指定了 storage_cooldown_time，则在到达 storage_cooldown_time 时间后，数据才会迁移。
	注意：当指定 storage_medium 时，如果 FE 参数 enable_strict_storage_medium_check 为False 该参数只是一个“尽力而为”的设置。即使集群内没有设置 SSD 存储介质，也不会报错，而是自动存储在可用的数据目录中。 同样，如果 SSD 介质不可访问、空间不足，都可能导致数据初始直接存储在其他可用介质上。而数据到期迁移到 HDD 时，如果 HDD 介质不 可 访 问 、 空 间 不 足 ， 也 可 能 迁 移 失 败 （ 但 是 会 不 断 尝 试 ） 。 如 果 FE 参 数enable_strict_storage_medium_check 为 True 则当集群内没有设置 SSD 存储介质时，会报错Failed to find enough host in all backends with storage medium is SSD。
~~~

##### 3.4.3.6、colocate_with

当需要使用 Colocation Join 功能时，使用这个参数设置 Colocation Group。

`"colocate_with" = "group1"`

##### 3.4.3.7、bloom_filter_columns

用户指定需要添加 Bloom Filter 索引的列名称列表。各个列的 Bloom Filter 索引是独立的，并不是组合索引。

`"bloom_filter_columns" = "k1, k2, k3"`

##### 3.4.3.8、in_memory

已弃用。只支持设置为'false'。

##### 3.4.3.9、compression

Doris 表的默认压缩方式是 LZ4。1.1版本后，支持将压缩方式指定为ZSTD以获得更高的压缩比。

`"compression"="zstd"`

##### 3.4.3.10、function_column.sequence_col

当使用 UNIQUE KEY 模型时，可以指定一个sequence列，当KEY列相同时，将按照 sequence 列进行 REPLACE(较大值替换较小值，否则无法替换)

`function_column.sequence_col`用来指定sequence列到表中某一列的映射，该列可以为整型和时间类型（DATE、DATETIME），创建后不能更改该列的类型。如果设置了`function_column.sequence_col`, `function_column.sequence_type`将被忽略。

`"function_column.sequence_col" = 'column_name'`

##### 3.4.3.11、function_column.sequence_type

当使用 UNIQUE KEY 模型时，可以指定一个sequence列，当KEY列相同时，将按照 sequence 列进行 REPLACE(较大值替换较小值，否则无法替换)

这里我们仅需指定顺序列的类型，支持时间类型或整型。Doris 会创建一个隐藏的顺序列。

`"function_column.sequence_type" = 'Date'`

##### 3.4.3.12、light_schema_change

是否使用light schema change优化。

如果设置成 `true`, 对于值列的加减操作，可以更快地，同步地完成。

`"light_schema_change" = 'true'`

该功能在 2.0.0 及之后版本默认开启。

##### 3.4.3.13、disable_auto_compaction

是否对这个表禁用自动compaction。

如果这个属性设置成 `true`, 后台的自动compaction进程会跳过这个表的所有tablet。

`"disable_auto_compaction" = "false"`

##### 3.4.3.14、enable_single_replica_compaction

是否对这个表开启单副本 compaction。

如果这个属性设置成 `true`, 这个表的 tablet 的所有副本只有一个 do compaction，其他的从该副本拉取 rowset

`"enable_single_replica_compaction" = "false"`

##### 3.4.3.15、enable_duplicate_without_keys_by_default

当配置为`true`时，如果创建表的时候没有指定Unique、Aggregate或Duplicate时，会默认创建一个没有排序列和前缀索引的Duplicate模型的表。

`"enable_duplicate_without_keys_by_default" = "false"`

##### 3.4.3.16、skip_write_index_on_load

是否对这个表开启数据导入时不写索引.

如果这个属性设置成 `true`, 数据导入的时候不写索引（目前仅对倒排索引生效），而是在compaction的时候延迟写索引。这样可以避免首次写入和compaction 重复写索引的CPU和IO资源消耗，提升高吞吐导入的性能。

`"skip_write_index_on_load" = "false"`

##### 3.4.3.17、compaction_policy

配置这个表的 compaction 的合并策略，仅支持配置为 time_series 或者 size_based

time_series: 当 rowset 的磁盘体积积攒到一定大小时进行版本合并。合并后的 rowset 直接晋升到 base compaction 阶段。在时序场景持续导入的情况下有效降低 compact 的写入放大率

此策略将使用 time_series_compaction 为前缀的参数调整 compaction 的执行

`"compaction_policy" = ""`

##### 3.4.3.18、time_series_compaction_goal_size_mbytes

compaction 的合并策略为 time_series 时，将使用此参数来调整每次 compaction 输入的文件的大小，输出的文件大小和输入相当

`"time_series_compaction_goal_size_mbytes" = "1024"`

##### 3.4.3.19、time_series_compaction_file_count_threshold

compaction 的合并策略为 time_series 时，将使用此参数来调整每次 compaction 输入的文件数量的最小值

一个 tablet 中，文件数超过该配置，就会触发 compaction

`"time_series_compaction_file_count_threshold" = "2000"`

##### 3.4.3.20、time_series_compaction_time_threshold_seconds

compaction 的合并策略为 time_series 时，将使用此参数来调整 compaction 的最长时间间隔，即长时间未执行过 compaction 时，就会触发一次 compaction，单位为秒

`"time_series_compaction_time_threshold_seconds" = "3600"`

##### 3.4.3.21、动态分区相关

动态分区相关参数如下：

- `dynamic_partition.enable`: 用于指定表级别的动态分区功能是否开启。默认为 true。
- `dynamic_partition.time_unit:` 用于指定动态添加分区的时间单位，可选择为DAY（天），WEEK(周)，MONTH（月），YEAR（年），HOUR（时）。
- `dynamic_partition.start`: 用于指定向前删除多少个分区。值必须小于0。默认为 Integer.MIN_VALUE。
- `dynamic_partition.end`: 用于指定提前创建的分区数量。值必须大于0。
- `dynamic_partition.prefix`: 用于指定创建的分区名前缀，例如分区名前缀为p，则自动创建分区名为p20200108。
- `dynamic_partition.buckets`: 用于指定自动创建的分区分桶数量。
- `dynamic_partition.create_history_partition`: 是否创建历史分区。
- `dynamic_partition.history_partition_num`: 指定创建历史分区的数量。
- `dynamic_partition.reserved_history_periods`: 用于指定保留的历史分区的时间段

#### 3.4.4、ENGINE

本示例中，ENGINE 的类型是 olap，即默认的 ENGINE 类型。在 Doris 中，只有这个 ENGINE 类型是由 Doris 负责数据管理和存储的。其他 ENGINE 类型，如 mysql、broker、es 等等，本质上只是对外部其他数据库或系统中的表的映射，以保证 Doris 可以读取这些数据。而 Doris 本身并不创建、管理和存储任何非 olap ENGINE 类型的表和数据。

#### 3.4.5、其他

`IF NOT EXISTS` 表示如果没有创建过该表，则创建。注意这里只判断表名是否存在，而不会判断新建表结构是否与已存在的表结构相同。所以如果存在一个同名但不同构的表，该命令也会返回成功，但并不代表已经创建了新的表和新的结构。

### 3.5、数据模型

Doris 的数据模型主要分为3类:

- Aggregate
- Unique
- Duplicate

#### 3.5.1、Aggregate 模型

表中的列按照是否设置了 `AggregationType`，分为 Key (维度列) 和 Value（指标列）。没有设置 `AggregationType` 的，如 `user_id`、`date`、`age` ... 等称为 **Key**，而设置了 `AggregationType` 的称为 **Value**。

当我们导入数据时，对于 Key 列相同的行会聚合成一行，而 Value 列会按照设置的`AggregationType` 进行聚合。 `AggregationType` 目前有以下几种聚合方式和agg_state：

1. SUM：求和，多行的 Value 进行累加。
2. REPLACE：替代，下一批数据中的 Value 会替换之前导入过的行中的 Value。
3. MAX：保留最大值。
4. MIN：保留最小值。
5. REPLACE_IF_NOT_NULL：非空值替换。和 REPLACE 的区别在于对于null值，不做替换。
6. HLL_UNION：HLL 类型的列的聚合方式，通过 HyperLogLog 算法聚合。
7. BITMAP_UNION：BIMTAP 类型的列的聚合方式，进行位图的并集聚合。
8. 如果这几种聚合方式无法满足需求，则可以选择使用agg_state类型。

数据的聚合，在 Doris 中有如下三个阶段发生：

1. 每一批次数据导入的 ETL 阶段。该阶段会在每一批次导入的数据内部进行聚合。
2. 底层 BE 进行数据 Compaction 的阶段。该阶段，BE 会对已导入的不同批次的数据进行进一步的聚合。
3. 数据查询阶段。在数据查询时，对于查询涉及到的数据，会进行对应的聚合。

数据在不同时间，可能聚合的程度不一致。比如一批数据刚导入时，可能还未与之前已存在的数据进行聚合。但是对于用户而言，用户**只能查询到**聚合后的数据。即不同的聚合程度对于用户查询而言是透明的。用户需始终认为数据以**最终的完成的聚合程度**存在，而**不应假设某些聚合还未发生**。

##### 3.5.1.1、示例1：导入数据聚合

**1）建表**

~~~sql
CREATE DATABASE IF NOT EXISTS example_db;

CREATE TABLE IF NOT EXISTS example_db.example_tbl_agg1
(
    `user_id` LARGEINT NOT NULL COMMENT "用户id",
    `date` DATE NOT NULL COMMENT "数据灌入日期时间",
    `city` VARCHAR(20) COMMENT "用户所在城市",
    `age` SMALLINT COMMENT "用户年龄",
    `sex` TINYINT COMMENT "用户性别",
    `last_visit_date` DATETIME REPLACE DEFAULT "1970-01-01 00:00:00" COMMENT "用户最后一次访问时间",
    `cost` BIGINT SUM DEFAULT "0" COMMENT "用户总消费",
    `max_dwell_time` INT MAX DEFAULT "0" COMMENT "用户最大停留时间",
    `min_dwell_time` INT MIN DEFAULT "99999" COMMENT "用户最小停留时间"
)
AGGREGATE KEY(`user_id`, `date`, `city`, `age`, `sex`)
DISTRIBUTED BY HASH(`user_id`) BUCKETS 1
PROPERTIES (
"replication_allocation" = "tag.location.default: 1"
);
~~~

**2）插入数据**

~~~sql
insert into example_db.example_tbl_agg1 values
(10000,"2017-10-01","北京",20,0,"2017-10-01 06:00:00",20,10,10),
(10000,"2017-10-01","北京",20,0,"2017-10-01 07:00:00",15,2,2),
(10001,"2017-10-01","北京",30,1,"2017-10-01 17:05:45",2,22,22),
(10002,"2017-10-02","上海",20,1,"2017-10-02 12:59:12",200,5,5),
(10003,"2017-10-02","广州",32,0,"2017-10-02 11:20:00",30,11,11),
(10004,"2017-10-01","深圳",35,0,"2017-10-01 10:00:15",100,3,3),
(10004,"2017-10-03","深圳",35,0,"2017-10-03 10:20:22",11,6,6);
~~~

注意：<font color="red">Insert into 单条数据这种操作在 Doris 里只能演示不能在生产使用，会引发写阻塞。</font>

**3）查看表**

~~~sql
select * from example_db.example_tbl_agg1;
~~~

可以看到，用户 10000 只剩下了一行**聚合后**的数据。而其余用户的数据和原始数据保持一致。经过聚合，Doris 中最终只会存储聚合后的数据。换句话说，即明细数据会丢失，用户不能够再查询到聚合前的明细数据了。

##### 3.5.1.2、示例2：保留明细数据

**1）建表**

~~~sql
CREATE TABLE IF NOT EXISTS example_db.example_tbl_agg2
(
    `user_id` LARGEINT NOT NULL COMMENT "用户id",
    `date` DATE NOT NULL COMMENT "数据灌入日期时间",
    `timestamp` DATETIME NOT NULL COMMENT "数据灌入日期时间戳",
    `city` VARCHAR(20) COMMENT "用户所在城市",
    `age` SMALLINT COMMENT "用户年龄",
    `sex` TINYINT COMMENT "用户性别",
    `last_visit_date` DATETIME REPLACE DEFAULT "1970-01-01 00:00:00" COMMENT "用户最后一次访问时间",
    `cost` BIGINT SUM DEFAULT "0" COMMENT "用户总消费",
    `max_dwell_time` INT MAX DEFAULT "0" COMMENT "用户最大停留时间",
    `min_dwell_time` INT MIN DEFAULT "99999" COMMENT "用户最小停留时间"
)
AGGREGATE KEY(`user_id`, `date`, `timestamp` ,`city`, `age`, `sex`)
DISTRIBUTED BY HASH(`user_id`) BUCKETS 1
PROPERTIES (
"replication_allocation" = "tag.location.default: 1"
);
~~~

**2）插入数据**

~~~sql
insert into example_db.example_tbl_agg2 values
(10000,"2017-10-01","2017-10-01 08:00:05","北京",20,0,"2017-10-01 06:00:00",20,10,10),
(10000,"2017-10-01","2017-10-01 09:00:05","北京",20,0,"2017-10-01 07:00:00",15,2,2),
(10001,"2017-10-01","2017-10-01 18:12:10","北京",30,1,"2017-10-01 17:05:45",2,22,22),
(10002,"2017-10-02","2017-10-02 13:10:00","上海",20,1,"2017-10-02 12:59:12",200,5,5),
(10003,"2017-10-02","2017-10-02 13:15:00","广州",32,0,"2017-10-02 11:20:00",30,11,11),
(10004,"2017-10-01","2017-10-01 12:12:48","深圳",35,0,"2017-10-01 10:00:15",100,3,3),
(10004,"2017-10-03","2017-10-03 12:38:20","深圳",35,0,"2017-10-03 10:20:22",11,6,6);
~~~

**3）查看表**

~~~sql
select * from example_db.example_tbl_agg2;
~~~

我们可以看到，存储的数据，和导入数据完全一样，没有发生任何聚合。这是因为，这批数据中，因为加入了 `timestamp` 列，所有行的 Key 都**不完全相同**。也就是说，只要保证导入的数据中，每一行的 Key 都不完全相同，那么即使在聚合模型下，Doris 也可以保存完整的明细数据。

##### 3.5.1.3、示例3：导入数据与已有数据聚合

**1）往示例1再次插入数据**

~~~sql
insert into example_db.example_tbl_agg1 values
(10004,"2017-10-03","深圳",35,0,"2017-10-03 11:22:00",44,19,19),
(10005,"2017-10-03","长沙",29,1,"2017-10-03 18:11:02",3,1,1);
~~~

**2）查看表**

~~~sql
select * from example_db.example_tbl_agg1;
~~~

可以看到，用户 10004 的已有数据和新导入的数据发生了聚合。同时新增了 10005 用户的数据。

#### 3.5.2、Unique 模型

在某些多维分析场景下，用户更关注的是如何保证 Key 的唯一性，即如何获得 Primary Key 唯一性约束。 因此，我们引入了 Unique 数据模型。在1.2版本之前，该模型本质上是聚合模型的一个特例，也是一种简化的表结构表示方式。 由于聚合模型的实现方式是读时合并（merge on read)，因此在一些聚合查询上性能不佳（参考后续章节[聚合模型的局限性](https://doris.apache.org/zh-CN/docs/data-table/data-model#聚合模型的局限性)的描述）， 在1.2版本我们引入了Unique模型新的实现方式，写时合并（merge on write），通过在写入时做一些额外的工作，实现了最优的查询性能。 写时合并将在未来替换读时合并成为Unique模型的默认实现方式，两者将会短暂的共存一段时间。下面将对两种实现方式分别举例进行说明。

##### 3.5.2.1、读时合并（与聚合模型相同的实现方式）

1）**建表**

~~~sql
-- 唯一（unique）模型结构
CREATE TABLE IF NOT EXISTS example_db.example_tbl_unique
(
    `user_id` LARGEINT NOT NULL COMMENT "用户id",
    `username` VARCHAR(50) NOT NULL COMMENT "用户昵称",
    `city` VARCHAR(20) COMMENT "用户所在城市",
    `age` SMALLINT COMMENT "用户年龄",
    `sex` TINYINT COMMENT "用户性别",
    `phone` LARGEINT COMMENT "用户电话",
    `address` VARCHAR(500) COMMENT "用户地址",
    `register_time` DATETIME COMMENT "用户注册时间"
)
UNIQUE KEY(`user_id`, `username`)
DISTRIBUTED BY HASH(`user_id`) BUCKETS 1
PROPERTIES (
"replication_allocation" = "tag.location.default: 1"
);
~~~

**而这个表结构，完全同等于以下使用聚合模型描述的表结构：**

~~~sql
-- 聚合（aggregate）模型结构
CREATE TABLE IF NOT EXISTS example_db.example_tbl_agg3
(
    `user_id` LARGEINT NOT NULL COMMENT "用户id",
    `username` VARCHAR(50) NOT NULL COMMENT "用户昵称",
    `city` VARCHAR(20) REPLACE COMMENT "用户所在城市",
    `age` SMALLINT REPLACE COMMENT "用户年龄",
    `sex` TINYINT REPLACE COMMENT "用户性别",
    `phone` LARGEINT REPLACE COMMENT "用户电话",
    `address` VARCHAR(500) REPLACE COMMENT "用户地址",
    `register_time` DATETIME REPLACE COMMENT "用户注册时间"
)
AGGREGATE KEY(`user_id`, `username`)
DISTRIBUTED BY HASH(`user_id`) BUCKETS 1
PROPERTIES (
"replication_allocation" = "tag.location.default: 1"
);
~~~

**2）插入数据**

~~~sql
insert into example_db.example_tbl_unique values\
(10000,'wuyanzu',' 北 京 ',18,0,12345678910,' 北 京 朝 阳 区 ','2017-10-01 
07:00:00'),\
(10000,'wuyanzu',' 北 京 ',19,0,12345678910,' 北 京 朝 阳 区 ','2017-10-01 
07:00:00'),\
(10000,'zhangsan','北京',20,0,12345678910,'北京海淀区','2017-11-15 
06:10:20');
~~~

**3）查看表**

~~~sql
select * from example_db.example_tbl_unique;
~~~

即Unique 模型的读时合并实现完全可以用聚合模型中的 REPLACE 方式替代。其内部的实现方式和数据存储方式也完全一样。

##### 3.5.2.2、写时合并

Unique模型的写时合并实现，与聚合模型就是完全不同的两种模型了，查询性能更接近于duplicate模型，在有主键约束需求的场景上相比聚合模型有较大的查询性能优势，尤其是在聚合查询以及需要用索引过滤大量数据的查询中。

在 1.2.0 版本中，作为一个新的feature，写时合并默认关闭，用户可以通过添加下面的property来开启

```text
"enable_unique_key_merge_on_write" = "true"
```

注意：

1. 建议使用1.2.4及以上版本，该版本修复了一些bug和稳定性问题
2. 在be.conf中添加配置项：`disable_storage_page_cache=false`。不添加该配置项可能会对数据导入性能产生较大影响

**1）建表**

~~~sql
CREATE TABLE IF NOT EXISTS example_db.example_tbl_unique_merge_on_write
(
    `user_id` LARGEINT NOT NULL COMMENT "用户id",
    `username` VARCHAR(50) NOT NULL COMMENT "用户昵称",
    `city` VARCHAR(20) COMMENT "用户所在城市",
    `age` SMALLINT COMMENT "用户年龄",
    `sex` TINYINT COMMENT "用户性别",
    `phone` LARGEINT COMMENT "用户电话",
    `address` VARCHAR(500) COMMENT "用户地址",
    `register_time` DATETIME COMMENT "用户注册时间"
)
UNIQUE KEY(`user_id`, `username`)
DISTRIBUTED BY HASH(`user_id`) BUCKETS 1
PROPERTIES (
"replication_allocation" = "tag.location.default: 1",
"enable_unique_key_merge_on_write" = "true"
);
~~~

**2）插入数据**

~~~sql
insert into example_db.example_tbl_unique_merge_on_write values\
(10000,'wuyanzu',' 北 京 ',18,0,12345678910,' 北 京 朝 阳 区 ','2017-10-01 
07:00:00'),\
(10000,'wuyanzu',' 北 京 ',19,0,12345678910,' 北 京 朝 阳 区 ','2017-10-01 
07:00:00'),\
(10000,'zhangsan','北京',20,0,12345678910,'北京海淀区','2017-11-15 
06:10:20');
~~~

**3）查看表**

~~~sql
select * from example_db.example_tbl_unique_merge_on_write;
~~~

在开启了写时合并选项的Unique表上，数据在导入阶段就会去将被覆盖和被更新的数据进行标记删除，同时将新的数据写入新的文件。在查询的时候， 所有被标记删除的数据都会在文件级别被过滤掉，读取出来的数据就都是最新的数据，消除掉了读时合并中的数据聚合过程，并且能够在很多情况下支持多种谓词的下推。因此在许多场景都能带来比较大的性能提升，尤其是在有聚合查询的情况下。

【注意】

1. 新的Merge-on-write实现默认关闭，且只能在建表时通过指定property的方式打开。
2. 旧的Merge-on-read的实现无法无缝升级到新版本的实现（数据组织方式完全不同），如果需要改为使用写时合并的实现版本，需要手动执行`insert into unique-mow-table select * from source table`.
3. 在Unique模型上独有的delete sign 和 sequence col，在写时合并的新版实现中仍可以正常使用，用法没有变化。

#### 3.5.3、Duplicate 模型

在某些多维分析场景下，数据既没有主键，也没有聚合需求。因此，我们引入 Duplicate 数据模型来满足这类需求。

**1）建表**

~~~~sql
CREATE TABLE IF NOT EXISTS example_db.example_tbl_duplicate
(
    `timestamp` DATETIME NOT NULL COMMENT "日志时间",
    `type` INT NOT NULL COMMENT "日志类型",
    `error_code` INT COMMENT "错误码",
    `error_msg` VARCHAR(1024) COMMENT "错误详细信息",
    `op_id` BIGINT COMMENT "负责人id",
    `op_time` DATETIME COMMENT "处理时间"
)
DUPLICATE KEY(`timestamp`, `type`, `error_code`)
DISTRIBUTED BY HASH(`type`) BUCKETS 1
PROPERTIES (
"replication_allocation" = "tag.location.default: 1"
);
~~~~

**2）插入数据**

~~~sql
insert into example_db.example_tbl_duplicate values\
('2017-10-01 08:00:05',1,404,'not found page', 101, '2017-10-01 
08:00:05'),\
('2017-10-01 08:00:05',1,404,'not found page', 101, '2017-10-01 
08:00:05'),\
('2017-10-01 08:00:05',2,404,'not found page', 101, '2017-10-01 
08:00:06'),\
('2017-10-01 08:00:06',2,404,'not found page', 101, '2017-10-01 
08:00:07');
~~~

**3）查看表**

~~~sql
select * from example_db.example_tbl_duplicate;
~~~

这种数据模型区别于 `Aggregate` 和 `Unique`模型。数据完全按照导入文件中的数据进行存储，不会有任何聚合。即使两行数据完全相同，也都会保留。 而在建表语句中指定的 `DUPLICATE KEY`，只是用来指明底层数据按照那些列进行排序。（更贴切的名称应该为 “`Sorted Column`”， 这里取名 “DUPLICATE KEY” 只是用以明确表示所用的数据模型。关于 “Sorted Column”的更多解释，可以参阅[前缀索引](https://doris.apache.org/zh-CN/docs/data-table/index/index-overview)）。在 DUPLICATE KEY 的选择上，我们建议适当的选择前 2-4 列就可以。

这种数据模型适用于既没有聚合需求，又没有主键唯一性约束的原始数据的存储。更多使用场景，可参阅[聚合模型的局限性](https://doris.apache.org/zh-CN/docs/data-table/data-model#聚合模型的局限性)小节。

##### 3.5.3.1、无排序列 Duplicate 模型

当创建表的时候没有指定Unique、Aggregate或Duplicate时，会默认创建一个Duplicate模型的表，并自动指定排序列。

当用户并没有排序需求的时候，可以通过在表属性中配置：

```text
"enable_duplicate_without_keys_by_default" = "true"
```

然后再创建默认模型的时候，就会不再指定排序列，也不会给该表创建前缀索引，以此减少在导入和存储上额外的开销。

**1）建表**

~~~sql
CREATE TABLE IF NOT EXISTS example_db.example_tbl_duplicate_without_keys_by_default
(
    `timestamp` DATETIME NOT NULL COMMENT "日志时间",
    `type` INT NOT NULL COMMENT "日志类型",
    `error_code` INT COMMENT "错误码",
    `error_msg` VARCHAR(1024) COMMENT "错误详细信息",
    `op_id` BIGINT COMMENT "负责人id",
    `op_time` DATETIME COMMENT "处理时间"
)
DISTRIBUTED BY HASH(`type`) BUCKETS 1
PROPERTIES (
"replication_allocation" = "tag.location.default: 1",
"enable_duplicate_without_keys_by_default" = "true"
);
~~~

**2）插入数据**

~~~sql
insert into example_db.example_tbl_duplicate_without_keys_by_default values\
('2017-10-01 08:00:05',1,404,'not found page', 101, '2017-10-01 
08:00:05'),\
('2017-10-01 08:00:05',1,404,'not found page', 101, '2017-10-01 
08:00:05'),\
('2017-10-01 08:00:05',2,404,'not found page', 101, '2017-10-01 
08:00:06'),\
('2017-10-01 08:00:06',2,404,'not found page', 101, '2017-10-01 
08:00:07');
~~~

**3）查看表**

~~~sql
select * from example_db.example_tbl_duplicate_without_keys_by_default;
~~~

#### 3.5.4、聚合模型的局限性

##### 3.5.4.1、Aggregate 模型

这里我们针对 Aggregate 模型，来介绍下聚合模型的局限性。

在聚合模型中，模型对外展现的，是**最终聚合后的**数据。也就是说，任何还未聚合的数据（比如说两个不同导入批次的数据），必须通过某种方式，以保证对外展示的一致性。我们举例说明。

假设表结构如下：

| ColumnName | Type     | AggregationType | Comment      |
| ---------- | -------- | --------------- | ------------ |
| user_id    | LARGEINT |                 | 用户id       |
| date       | DATE     |                 | 数据灌入日期 |
| cost       | BIGINT   | SUM             | 用户总消费   |

假设存储引擎中有如下两个已经导入完成的批次的数据：

**batch 1**

| user_id | date       | cost |
| ------- | ---------- | ---- |
| 10001   | 2017-11-20 | 50   |
| 10002   | 2017-11-21 | 39   |

**batch 2**

| user_id | date       | cost |
| ------- | ---------- | ---- |
| 10001   | 2017-11-20 | 1    |
| 10001   | 2017-11-21 | 5    |
| 10003   | 2017-11-22 | 22   |

可以看到，用户 10001 分属在两个导入批次中的数据还没有聚合。但是为了保证用户只能查询到如下最终聚合后的数据：

| user_id | date       | cost |
| ------- | ---------- | ---- |
| 10001   | 2017-11-20 | 51   |
| 10001   | 2017-11-21 | 5    |
| 10002   | 2017-11-21 | 39   |
| 10003   | 2017-11-22 | 22   |

我们在查询引擎中加入了聚合算子，来保证数据对外的一致性。

另外，在聚合列（Value）上，执行与聚合类型不一致的聚合类查询时，要注意语意。比如我们在如上示例中执行如下查询：

```text
SELECT MIN(cost) FROM table;
```

得到的结果是 5，而不是 1。

同时，这种一致性保证，在某些查询中，会极大地降低查询效率。

我们以最基本的 count(*) 查询为例：

```text
SELECT COUNT(*) FROM table;
```

在其他数据库中，这类查询都会很快地返回结果。因为在实现上，我们可以通过如“导入时对行进行计数，保存 count 的统计信息”，或者在查询时“仅扫描某一列数据， 获得 count 值”的方式，只需很小的开销，即可获得查询结果。但是在 Doris 的聚合模型中，这种查询的开销**非常大**。

我们以刚才的数据为例：

**batch 1**

| user_id | date       | cost |
| ------- | ---------- | ---- |
| 10001   | 2017-11-20 | 50   |
| 10002   | 2017-11-21 | 39   |

**batch 2**

| user_id | date       | cost |
| ------- | ---------- | ---- |
| 10001   | 2017-11-20 | 1    |
| 10001   | 2017-11-21 | 5    |
| 10003   | 2017-11-22 | 22   |

因为最终的聚合结果为：

| user_id | date       | cost |
| ------- | ---------- | ---- |
| 10001   | 2017-11-20 | 51   |
| 10001   | 2017-11-21 | 5    |
| 10002   | 2017-11-21 | 39   |
| 10003   | 2017-11-22 | 22   |

所以，`select count(*) from table;` 的正确结果应该为 **4**。但如果我们只扫描 `user_id` 这一列，如果加上查询时聚合，最终得到的结果是 **3**（10001, 10002, 10003）。而如果不加查询时聚合，则得到的结果是 **5**（两批次一共5行数据）。可见这两个结果都是不对的。

为了得到正确的结果，我们必须同时读取 `user_id` 和 `date` 这两列的数据，**再加上查询时聚合**，才能返回 **4** 这个正确的结果。也就是说，在 count(*) 查询中，Doris 必须扫描所有的 AGGREGATE KEY 列（这里就是 `user_id` 和 `date`），并且聚合后，才能得到语意正确的结果。当聚合列非常多时，count(*) 查询需要扫描大量的数据。

因此，当业务上有频繁的 count(*) 查询时，我们建议用户通过增加一个**值恒为 1 的，聚合类型为 SUM 的列来模拟 count(\*)**。如刚才的例子中的表结构，我们修改如下：

| ColumnName | Type   | AggregateType | Comment       |
| ---------- | ------ | ------------- | ------------- |
| user_id    | BIGINT |               | 用户id        |
| date       | DATE   |               | 数据灌入日期  |
| cost       | BIGINT | SUM           | 用户总消费    |
| count      | BIGINT | SUM           | 用于计算count |

增加一个 count 列，并且导入数据中，该列值**恒为 1**。则 `select count(*) from table;` 的结果等价于 `select sum(count) from table;`。 而后者的查询效率将远高于前者。不过这种方式也有使用限制，就是用户需要自行保证，不会重复导入 AGGREGATE KEY 列都相同地行。 否则，`select sum(count) from table;` 只能表述原始导入的行数，而不是 `select count(*) from table;` 的语义。

另一种方式，就是 **将如上的 `count` 列的聚合类型改为 REPLACE，且依然值恒为 1**。那么 `select sum(count) from table;` 和 `select count(*) from table;` 的结果将是一致的。并且这种方式，没有导入重复行的限制。

##### 3.5.4.2、Unique模型的写时合并实现

Unique模型的写时合并实现没有聚合模型的局限性，还是以刚才的数据为例，写时合并为每次导入的rowset增加了对应的delete bitmap，来标记哪些数据被覆盖。第一批数据导入后状态如下

**batch 1**

| user_id | date       | cost | delete bit |
| ------- | ---------- | ---- | ---------- |
| 10001   | 2017-11-20 | 50   | false      |
| 10002   | 2017-11-21 | 39   | false      |

当第二批数据导入完成后，第一批数据中重复的行就会被标记为已删除，此时两批数据状态如下

**batch 1**

| user_id | date       | cost | delete bit |
| ------- | ---------- | ---- | ---------- |
| 10001   | 2017-11-20 | 50   | **true**   |
| 10002   | 2017-11-21 | 39   | false      |

**batch 2**

| user_id | date       | cost | delete bit |
| ------- | ---------- | ---- | ---------- |
| 10001   | 2017-11-20 | 1    | false      |
| 10001   | 2017-11-21 | 5    | false      |
| 10003   | 2017-11-22 | 22   | false      |

在查询时，所有在delete bitmap中被标记删除的数据都不会读出来，因此也无需进行做任何数据聚合，上述数据中有效地行数为4行， 查询出的结果也应该是4行，也就可以采取开销最小的方式来获取结果，即前面提到的“仅扫描某一列数据，获得 count 值”的方式。

在测试环境中，count(*) 查询在Unique模型的写时合并实现上的性能，相比聚合模型有10倍以上的提升。

##### 3.5.4.3、Duplicate 模型

Duplicate 模型没有聚合模型的这个局限性。因为该模型不涉及聚合语意，在做 count(*) 查询时，任意选择一列查询，即可得到语意正确的结果。

### 3.6、动态分区

动态分区是在 Doris 0.12 版本中引入的新功能。旨在对表级别的分区实现生命周期管理(TTL)，减少用户的使用负担。

目前实现了动态添加分区及动态删除分区的功能。

动态分区只支持 Range 分区。

注意：这个功能在被CCR同步时将会失效。如果这个表是被CCR复制而来的，即PROPERTIES中包含`is_being_synced = true`时，在`show create table`中会显示开启状态，但不会实际生效。当`is_being_synced`被设置为 `false` 时，这些功能将会恢复生效，但`is_being_synced`属性仅供CCR外围模块使用，在CCR同步的过程中不要手动设置。

#### 3.6.1、原理

在某些使用场景下，用户会将表按照天进行分区划分，每天定时执行例行任务，这时需要使用方手动管理分区，否则可能由于使用方没有创建分区导致数据导入失败，这给使用方带来了额外的维护成本。

通过动态分区功能，用户可以在建表时设定动态分区的规则。FE 会启动一个后台线程，根据用户指定的规则创建或删除分区。用户也可以在运行时对现有规则进行变更。

#### 3.6.2、使用方式

动态分区的规则可以在建表时指定，或者在运行时进行修改。当前仅支持对单分区列的分区表设定动态分区规则。

- 建表时指定：

    ```sql
    CREATE TABLE tbl1
    (...)
    PROPERTIES
    (
        "dynamic_partition.prop1" = "value1",
        "dynamic_partition.prop2" = "value2",
        ...
    )
    ```

- 运行时修改

    ```sql
    ALTER TABLE tbl1 SET
    (
        "dynamic_partition.prop1" = "value1",
        "dynamic_partition.prop2" = "value2",
        ...
    )
    ```

#### 3.6.3、动态分区规则参数

##### 3.6.3.1、主要参数

动态分区的规则参数都以 `dynamic_partition.` 为前缀：

| 参数名                                    | 说明                                                         |
| ----------------------------------------- | ------------------------------------------------------------ |
| `dynamic_partition.enable`                | 是否开启动态分区特性。可指定为 `TRUE` 或 `FALSE`。如果不填写，默认为 `TRUE`。如果为 `FALSE`，则 Doris 会忽略该表的动态分区规则。 |
| `dynamic_partition.time_unit`（必选参数） | 动态分区调度的单位。可指定为 `HOUR`、`DAY`、`WEEK`、`MONTH`、`YEAR`。<br />分别表示按小时、按天、按星期、按月、按年进行分区创建或删除。<br />当指定为 `HOUR` 时，后缀格式为 `yyyyMMddHH`，小时为单位的分区列数据类型不能为 DATE。<br />当指定为 `DAY` 时，后缀格式为 `yyyyMMdd`。<br />当指定为 `WEEK` 时，后缀格式为`yyyy_ww`。即当前日期属于这一年的第几周。<br />当指定为 `MONTH` 时，后缀格式为 `yyyyMM`。<br />当指定为 `YEAR` 时，后缀格式为 `yyyy`。 |
| `dynamic_partition.time_zone`             | 动态分区的时区，如果不填写，则默认为当前机器的系统的时区，例如 `Asia/Shanghai`，如果想获取当前支持的时区设置，可以参考 `https://en.wikipedia.org/wiki/List_of_tz_database_time_zones`。 |
| `dynamic_partition.start`                 | 动态分区的起始偏移，为负数。根据 `time_unit` 属性的不同，以当天（星期/月）为基准，分区范围在此偏移之前的分区将会被删除。如果不填写，则默认为 `-2147483648`，即不删除历史分区。 |
| `dynamic_partition.end`（必选参数）       | 动态分区的结束偏移，为正数。根据 `time_unit` 属性的不同，以当天（星期/月）为基准，提前创建对应范围的分区。 |
| `dynamic_partition.prefix`（必选参数）    | 动态创建的分区名前缀。                                       |
| `dynamic_partition.buckets`               | 动态创建的分区所对应的分桶数量。                             |
| `dynamic_partition.replication_num`       | 动态创建的分区所对应的副本数量，如果不填写，则默认为该表创建时指定的副本数量。 |
| `dynamic_partition.start_day_of_week`     | 当 `time_unit` 为 `WEEK` 时，该参数用于指定每周的起始点。取值为 1 到 7。其中 1 表示周一，7 表示周日。默认为 1，即表示每周以周一为起始点。 |
| `dynamic_partition.start_day_of_month`    | 当 `time_unit` 为 `MONTH` 时，该参数用于指定每月的起始日期。取值为 1 到 28。其中 1 表示每月1号，28 表示每月28号。默认为 1，即表示每月以1号位起始点。暂不支持以29、30、31号为起始日，以避免因闰年或闰月带来的歧义。 |

##### 3.6.3.2、创建历史分区的参数

- `dynamic_partition.create_history_partition`

    默认为 false。当置为 true 时，Doris 会自动创建所有分区，具体创建规则见下文。同时，FE 的参数 `max_dynamic_partition_num` 会限制总分区数量，以避免一次性创建过多分区。当期望创建的分区个数大于 `max_dynamic_partition_num` 值时，操作将被禁止。

    当不指定 `start` 属性时，该参数不生效。

- `dynamic_partition.history_partition_num`

    当 `create_history_partition` 为 `true` 时，该参数用于指定创建历史分区数量。默认值为 -1， 即未设置。

- `dynamic_partition.hot_partition_num`

    指定最新的多少个分区为热分区。对于热分区，系统会自动设置其 `storage_medium` 参数为SSD，并且设置 `storage_cooldown_time`。

    **注意：若存储路径下没有 SSD 磁盘路径，配置该参数会导致动态分区创建失败。**

    `hot_partition_num` 是往前 n 天和未来所有分区

    我们举例说明。假设今天是 2021-05-20，按天分区，动态分区的属性设置为：hot_partition_num=2, end=3, start=-3。则系统会自动创建以下分区，并且设置 `storage_medium` 和 `storage_cooldown_time` 参数：

    ```text
    p20210517：["2021-05-17", "2021-05-18") storage_medium=HDD storage_cooldown_time=9999-12-31 23:59:59
    p20210518：["2021-05-18", "2021-05-19") storage_medium=HDD storage_cooldown_time=9999-12-31 23:59:59
    p20210519：["2021-05-19", "2021-05-20") storage_medium=SSD storage_cooldown_time=2021-05-21 00:00:00
    p20210520：["2021-05-20", "2021-05-21") storage_medium=SSD storage_cooldown_time=2021-05-22 00:00:00
    p20210521：["2021-05-21", "2021-05-22") storage_medium=SSD storage_cooldown_time=2021-05-23 00:00:00
    p20210522：["2021-05-22", "2021-05-23") storage_medium=SSD storage_cooldown_time=2021-05-24 00:00:00
    p20210523：["2021-05-23", "2021-05-24") storage_medium=SSD storage_cooldown_time=2021-05-25 00:00:00
    ```

    

- `dynamic_partition.reserved_history_periods`

    需要保留的历史分区的时间范围。当`dynamic_partition.time_unit` 设置为 "DAY/WEEK/MONTH/YEAR" 时，需要以 `[yyyy-MM-dd,yyyy-MM-dd],[...,...]` 格式进行设置。当`dynamic_partition.time_unit` 设置为 "HOUR" 时，需要以 `[yyyy-MM-dd HH:mm:ss,yyyy-MM-dd HH:mm:ss],[...,...]` 的格式来进行设置。如果不设置，默认为 `"NULL"`。

    我们举例说明。假设今天是 2021-09-06，按天分类，动态分区的属性设置为：

    `time_unit="DAY/WEEK/MONTH/YEAR", end=3, start=-3, reserved_history_periods="[2020-06-01,2020-06-20],[2020-10-31,2020-11-15]"`。

    则系统会自动保留：

    ```text
    ["2020-06-01","2020-06-20"],
    ["2020-10-31","2020-11-15"]
    ```

    或者

    `time_unit="HOUR", end=3, start=-3, reserved_history_periods="[2020-06-01 00:00:00,2020-06-01 03:00:00]"`.

    则系统会自动保留：

    ```text
    ["2020-06-01 00:00:00","2020-06-01 03:00:00"]
    ```

    这两个时间段的分区。其中，`reserved_history_periods` 的每一个 `[...,...]` 是一对设置项，两者需要同时被设置，且第一个时间不能大于第二个时间。

- `dynamic_partition.storage_medium`

    指定创建的动态分区的默认存储介质。默认是 HDD，可选择 SSD。

    注意，当设置为SSD时，`hot_partition_num` 属性将不再生效，所有分区将默认为 SSD 存储介质并且冷却时间为 9999-12-31 23:59:59。

##### 3.6.3.3、**创建历史分区规则**

当 `create_history_partition` 为 `true`，即开启创建历史分区功能时，Doris 会根据 `dynamic_partition.start` 和 `dynamic_partition.history_partition_num` 来决定创建历史分区的个数。

假设需要创建的历史分区数量为 `expect_create_partition_num`，根据不同的设置具体数量如下：

1. `create_history_partition` = `true`
    - `dynamic_partition.history_partition_num` 未设置，即 -1. `expect_create_partition_num` = `end` - `start`;
    - `dynamic_partition.history_partition_num` 已设置 `expect_create_partition_num` = `end` - max(`start`, `-histoty_partition_num`);
2. `create_history_partition` = `false` 不会创建历史分区，`expect_create_partition_num` = `end` - 0;

当 `expect_create_partition_num` 大于 `max_dynamic_partition_num`（默认500）时，禁止创建过多分区。

##### 3.6.3.4、**创建历史举例说明**

1. 假设今天是 2021-05-20，按天分区，动态分区的属性设置为：`create_history_partition=true, end=3, start=-3, history_partition_num=1`，则系统会自动创建以下分区：

    ```text
    p20210519
    p20210520
    p20210521
    p20210522
    p20210523
    ```

2. `history_partition_num=5`，其余属性与 1 中保持一直，则系统会自动创建以下分区：

    ```text
    p20210517
    p20210518
    p20210519
    p20210520
    p20210521
    p20210522
    p20210523
    ```

3. `history_partition_num=-1` 即不设置历史分区数量，其余属性与 1 中保持一直，则系统会自动创建以下分区：

    ```text
    p20210517
    p20210518
    p20210519
    p20210520
    p20210521
    p20210522
    p20210523
    ```

##### 3.6.3.5、注意事项

动态分区使用过程中，如果因为一些意外情况导致 `dynamic_partition.start` 和 `dynamic_partition.end` 之间的某些分区丢失，那么当前时间与 `dynamic_partition.end` 之间的丢失分区会被重新创建，`dynamic_partition.start`与当前时间之间的丢失分区不会重新创建。

##### 3.6.3.6、修改动态分区属性

通过如下命令可以修改动态分区的属性：

```sql
ALTER TABLE tbl1 SET
(
    "dynamic_partition.prop1" = "value1",
    ...
);
```

某些属性的修改可能会产生冲突。假设之前分区粒度为 DAY，并且已经创建了如下分区：

```text
p20200519: ["2020-05-19", "2020-05-20")
p20200520: ["2020-05-20", "2020-05-21")
p20200521: ["2020-05-21", "2020-05-22")
```

如果此时将分区粒度改为 MONTH，则系统会尝试创建范围为 `["2020-05-01", "2020-06-01")` 的分区，而该分区的分区范围和已有分区冲突，所以无法创建。而范围为 `["2020-06-01", "2020-07-01")` 的分区可以正常创建。因此，2020-05-22 到 2020-05-30 时间段的分区，需要自行填补。

##### 3.6.3.7、示例

1. 表 tbl1 分区列 k1 类型为 DATE，创建一个动态分区规则。按天分区，只保留最近7天的分区，并且预先创建未来3天的分区。

    ```sql
    CREATE TABLE example_db.student_dynamic_partition1
    (
        id int,
        create_time DATE,
        name varchar(50),
        age int
    )
    duplicate key(id,create_time)
    PARTITION BY RANGE(create_time) ()
    DISTRIBUTED BY HASH(id) buckets 10
    PROPERTIES
    (
        "dynamic_partition.enable" = "true",
        "dynamic_partition.time_unit" = "DAY",
        "dynamic_partition.start" = "-7",
        "dynamic_partition.end" = "3",
        "dynamic_partition.prefix" = "p",
        "dynamic_partition.buckets" = "10",
        "replication_num" = "1"
    );
    ```

    假设当前日期为 2020-05-29。则根据以上规则，tbl1 会产生以下分区：

    ```text
    p20200529: ["2020-05-29", "2020-05-30")
    p20200530: ["2020-05-30", "2020-05-31")
    p20200531: ["2020-05-31", "2020-06-01")
    p20200601: ["2020-06-01", "2020-06-02")
    ```

    在第二天，即 2020-05-30，会创建新的分区 `p20200602: ["2020-06-02", "2020-06-03")`

    在 2020-06-06 时，因为 `dynamic_partition.start` 设置为 7，则将删除7天前的分区，即删除分区 `p20200529`。

2. 表 tbl1 分区列 k1 类型为 DATETIME，创建一个动态分区规则。按星期分区，只保留最近2个星期的分区，并且预先创建未来2个星期的分区。

    ```sql
    CREATE TABLE tbl1
    (
        k1 DATETIME,
        ...
    )
    PARTITION BY RANGE(k1) ()
    DISTRIBUTED BY HASH(k1)
    PROPERTIES
    (
        "dynamic_partition.enable" = "true",
        "dynamic_partition.time_unit" = "WEEK",
        "dynamic_partition.start" = "-2",
        "dynamic_partition.end" = "2",
        "dynamic_partition.prefix" = "p",
        "dynamic_partition.buckets" = "8"
    );
    ```

    假设当前日期为 2020-05-29，是 2020 年的第 22 周。默认每周起始为星期一。则根于以上规则，tbl1 会产生以下分区：

    ```text
    p2020_22: ["2020-05-25 00:00:00", "2020-06-01 00:00:00")
    p2020_23: ["2020-06-01 00:00:00", "2020-06-08 00:00:00")
    p2020_24: ["2020-06-08 00:00:00", "2020-06-15 00:00:00")
    ```

    其中每个分区的起始日期为当周的周一。同时，因为分区列 k1 的类型为 DATETIME，则分区值会补全时分秒部分，且皆为 0。

    在 2020-06-15，即第25周时，会删除2周前的分区，即删除 `p2020_22`。

    在上面的例子中，假设用户指定了周起始日为 `"dynamic_partition.start_day_of_week" = "3"`，即以每周三为起始日。则分区如下：

    ```text
    p2020_22: ["2020-05-27 00:00:00", "2020-06-03 00:00:00")
    p2020_23: ["2020-06-03 00:00:00", "2020-06-10 00:00:00")
    p2020_24: ["2020-06-10 00:00:00", "2020-06-17 00:00:00")
    ```

    即分区范围为当周的周三到下周的周二。

    - 注：2019-12-31 和 2020-01-01 在同一周内，如果分区的起始日期为 2019-12-31，则分区名为 `p2019_53`，如果分区的起始日期为 2020-01-01，则分区名为 `p2020_01`。

3. 表 tbl1 分区列 k1 类型为 DATE，创建一个动态分区规则。按月分区，不删除历史分区，并且预先创建未来2个月的分区。同时设定以每月3号为起始日。

    ```sql
    CREATE TABLE tbl1
    (
        k1 DATE,
        ...
    )
    PARTITION BY RANGE(k1) ()
    DISTRIBUTED BY HASH(k1)
    PROPERTIES
    (
        "dynamic_partition.enable" = "true",
        "dynamic_partition.time_unit" = "MONTH",
        "dynamic_partition.end" = "2",
        "dynamic_partition.prefix" = "p",
        "dynamic_partition.buckets" = "8",
        "dynamic_partition.start_day_of_month" = "3"
    );
    ```

    假设当前日期为 2020-05-29。则根于以上规则，tbl1 会产生以下分区：

    ```text
    p202005: ["2020-05-03", "2020-06-03")
    p202006: ["2020-06-03", "2020-07-03")
    p202007: ["2020-07-03", "2020-08-03")
    ```

    因为没有设置 `dynamic_partition.start`，则不会删除历史分区。

    假设今天为 2020-05-20，并设置以每月28号为起始日，则分区范围为：

    ```text
    p202004: ["2020-04-28", "2020-05-28")
    p202005: ["2020-05-28", "2020-06-28")
    p202006: ["2020-06-28", "2020-07-28")
    ```

4. 查看动态分区表调度情况

    通过以下命令可以进一步查看当前数据库下，所有动态分区表的调度情况：

    ```sql
    show dynamic partition tables;
    ```

    - LastUpdateTime: 最后一次修改动态分区属性的时间
    - LastSchedulerTime: 最后一次执行动态分区调度的时间
    - State: 最后一次执行动态分区调度的状态
    - LastCreatePartitionMsg: 最后一次执行动态添加分区调度的错误信息
    - LastDropPartitionMsg: 最后一次执行动态删除分区调度的错误信息

5. 查看表的分区

    ~~~sql
    SHOW PARTITIONS FROM example_db.student_dynamic_partition1;
    ~~~

6. 插入测试数据,可以全部成功（修改成对应时间）

    ~~~sql
    insert into example_db.student_dynamic_partition1 values(1,'2023-11-23 11:00:00','name1',18);
    insert into example_db.student_dynamic_partition1 values(1,'2023-11-24 11:00:00','name1',18);
    insert into example_db.student_dynamic_partition1 values(1,'2023-11-25 11:00:00','name1',18);
    ~~~

7. 设置创建历史分区

    ~~~sql
    ALTER TABLE example_db.student_dynamic_partition1 SET ("dynamic_partition.create_history_partition" = "true");
    -- 查看分区情况
    SHOW PARTITIONS FROM example_db.student_dynamic_partition1;
    ~~~

8. 动态分区表与手动分区表相互转换

    对于一个表来说，动态分区和手动分区可以自由转换，但二者不能同时存在，有且只有一种状态。

    **手动分区转换为动态分区**

    ​		如果一个表在创建时未指定动态分区，可以通过 `ALTER TABLE` 在运行时修改动态分区相关属性来转化为动态分区，具体示例可以通过 `HELP ALTER TABLE` 查看。

    ​		开启动态分区功能后，Doris 将不再允许用户手动管理分区，会根据动态分区属性来自动管理分区。

    **注意**：如果已设定 `dynamic_partition.start`，分区范围在动态分区起始偏移之前的历史分区将会被删除。

    **动态分区转换为手动分区**

    ​		通过执行 `ALTER TABLE tbl_name SET ("dynamic_partition.enable" = "false")` 即可关闭动态分区功能，将其转换为手动分区表。

    ​		关闭动态分区功能后，Doris 将不再自动管理分区，需要用户手动通过 `ALTER TABLE` 的方式创建或删除分区。

### 3.7、Rollup与查询

ROLLUP 在多维分析中是“上卷”的意思，即将数据按某种指定的粒度进行进一步聚合。

#### 3.7.1、基本概念

在 Doris 中，我们将用户通过建表语句创建出来的表称为 Base 表（Base Table）。Base 表中保存着按用户建表语句指定的方式存储的基础数据。

在 Base 表之上，我们可以创建任意多个 ROLLUP 表。这些 ROLLUP 的数据是基于 Base 表产生的，并且在物理上是**独立存储**的。

ROLLUP 表的基本作用，在于在 Base 表的基础上，获得更粗粒度的聚合数据。

下面我们用示例详细说明在不同数据模型中的 ROLLUP 表及其作用。

#### 3.7.2、Aggregate 和 Unique 模型中的 ROLLUP

因为 Unique 只是 Aggregate 模型的一个特例，所以这里我们不加以区别。

##### 3.7.2.1、示例1：获得每个用户的总消费

以3.5.1.2小节创建的表example_db.example_tbl_agg2为例

1）查看表结构

~~~sql
mysql> desc example_db.example_tbl_agg2 all;
+------------------+---------------+-----------------+-------------+---------------+------+-------+---------------------+---------+---------+------------+-------------+
| IndexName        | IndexKeysType | Field           | Type        | InternalType  | Null | Key   | Default             | Extra   | Visible | DefineExpr | WhereClause |
+------------------+---------------+-----------------+-------------+---------------+------+-------+---------------------+---------+---------+------------+-------------+
| example_tbl_agg2 | AGG_KEYS      | user_id         | LARGEINT    | LARGEINT      | No   | true  | NULL                |         | true    |            |             |
|                  |               | date            | DATE        | DATEV2        | No   | true  | NULL                |         | true    |            |             |
|                  |               | timestamp       | DATETIME    | DATETIMEV2(0) | No   | true  | NULL                |         | true    |            |             |
|                  |               | city            | VARCHAR(20) | VARCHAR(20)   | Yes  | true  | NULL                |         | true    |            |             |
|                  |               | age             | SMALLINT    | SMALLINT      | Yes  | true  | NULL                |         | true    |            |             |
|                  |               | sex             | TINYINT     | TINYINT       | Yes  | true  | NULL                |         | true    |            |             |
|                  |               | last_visit_date | DATETIME    | DATETIMEV2(0) | Yes  | false | 1970-01-01 00:00:00 | REPLACE | true    |            |             |
|                  |               | cost            | BIGINT      | BIGINT        | Yes  | false | 0                   | SUM     | true    |            |             |
|                  |               | max_dwell_time  | INT         | INT           | Yes  | false | 0                   | MAX     | true    |            |             |
|                  |               | min_dwell_time  | INT         | INT           | Yes  | false | 99999               | MIN     | true    |            |             |
+------------------+---------------+-----------------+-------------+---------------+------+-------+---------------------+---------+---------+------------+-------------+
10 rows in set (0.01 sec)
~~~

2）比如需要查看某个用户的总消费，那么可以建立一个只有 user_id 和 cost 的 rollup

~~~sql
alter table example_db.example_tbl_agg2 add rollup rollup_user_cost(user_id,cost);
~~~

3）再次查看表结构，可以看到生成了rollup_user_cost

~~~sql
mysql> desc example_db.example_tbl_agg2 all;
+------------------+---------------+-----------------+-------------+---------------+------+-------+---------------------+---------+---------+------------+-------------+
| IndexName        | IndexKeysType | Field           | Type        | InternalType  | Null | Key   | Default             | Extra   | Visible | DefineExpr | WhereClause |
+------------------+---------------+-----------------+-------------+---------------+------+-------+---------------------+---------+---------+------------+-------------+
| example_tbl_agg2 | AGG_KEYS      | user_id         | LARGEINT    | LARGEINT      | No   | true  | NULL                |         | true    |            |             |
|                  |               | date            | DATE        | DATEV2        | No   | true  | NULL                |         | true    |            |             |
|                  |               | timestamp       | DATETIME    | DATETIMEV2(0) | No   | true  | NULL                |         | true    |            |             |
|                  |               | city            | VARCHAR(20) | VARCHAR(20)   | Yes  | true  | NULL                |         | true    |            |             |
|                  |               | age             | SMALLINT    | SMALLINT      | Yes  | true  | NULL                |         | true    |            |             |
|                  |               | sex             | TINYINT     | TINYINT       | Yes  | true  | NULL                |         | true    |            |             |
|                  |               | last_visit_date | DATETIME    | DATETIMEV2(0) | Yes  | false | 1970-01-01 00:00:00 | REPLACE | true    |            |             |
|                  |               | cost            | BIGINT      | BIGINT        | Yes  | false | 0                   | SUM     | true    |            |             |
|                  |               | max_dwell_time  | INT         | INT           | Yes  | false | 0                   | MAX     | true    |            |             |
|                  |               | min_dwell_time  | INT         | INT           | Yes  | false | 99999               | MIN     | true    |            |             |
|                  |               |                 |             |               |      |       |                     |         |         |            |             |
| rollup_user_cost | AGG_KEYS      | user_id         | LARGEINT    | LARGEINT      | No   | true  | NULL                |         | true    |            |             |
|                  |               | cost            | BIGINT      | BIGINT        | Yes  | false | 0                   | SUM     | true    |            |             |
+------------------+---------------+-----------------+-------------+---------------+------+-------+---------------------+---------+---------+------------+-------------+
13 rows in set (0.00 sec)
~~~

4）然后可以通过 explain 查看执行计划，是否使用到了 rollup

~~~sql
mysql> explain select user_id,sum(cost) from example_db.example_tbl_agg2 group by user_id;
+-----------------------------------------------------------------------------------------------+
| Explain String                                                                                |
+-----------------------------------------------------------------------------------------------+
| PLAN FRAGMENT 0                                                                               |
|   OUTPUT EXPRS:                                                                               |
|     user_id[#14]                                                                              |
|     sum(cost)[#15]                                                                            |
|   PARTITION: UNPARTITIONED                                                                    |
|                                                                                               |
|   VRESULT SINK                                                                                |
|                                                                                               |
|   2:VEXCHANGE                                                                                 |
|      offset: 0                                                                                |
|                                                                                               |
| PLAN FRAGMENT 1                                                                               |
|                                                                                               |
|   PARTITION: HASH_PARTITIONED: mv_user_id[#0]                                                 |
|                                                                                               |
|   STREAM DATA SINK                                                                            |
|     EXCHANGE ID: 02                                                                           |
|     UNPARTITIONED                                                                             |
|                                                                                               |
|   1:VAGGREGATE (update finalize)                                                              |
|   |  output: sum(mva_SUM__cost[#1])[#13]                                                      |
|   |  group by: mv_user_id[#0]                                                                 |
|   |  cardinality=1                                                                            |
|   |  projections: mv_user_id[#12], sum(mva_SUM__cost)[#13]                                    |
|   |  project output tuple id: 3                                                               |
|   |                                                                                           |
|   0:VOlapScanNode                                                                             |
|      TABLE: default_cluster:example_db.example_tbl_agg2(rollup_user_cost), PREAGGREGATION: ON |
|      partitions=1/1, tablets=1/1, tabletList=12353                                            |
|      cardinality=7, avgRowSize=245.71428, numNodes=1                                          |
|      pushAggOp=NONE                                                                           |
+-----------------------------------------------------------------------------------------------+
31 rows in set (0.02 sec)
~~~

Doris 会自动命中这个 ROLLUP 表，从而只需扫描极少的数据量，即可完成这次聚合查询。

5）可以通过下述命令查看rollup的信息

~~~sql
mysql> SHOW ALTER TABLE ROLLUP;
+-------+------------------+---------------------+---------------------+------------------+------------------+----------+---------------+----------+------+----------+---------+
| JobId | TableName        | CreateTime          | FinishTime          | BaseIndexName    | RollupIndexName  | RollupId | TransactionId | State    | Msg  | Progress | Timeout |
+-------+------------------+---------------------+---------------------+------------------+------------------+----------+---------------+----------+------+----------+---------+
| 12351 | example_tbl_agg2 | 2023-11-23 15:20:20 | 2023-11-23 15:20:21 | example_tbl_agg2 | rollup_user_cost | 12352    | 2073          | FINISHED |      | NULL     | 2592000 |
+-------+------------------+---------------------+---------------------+------------------+------------------+----------+---------------+----------+------+----------+---------+
1 row in set (0.01 sec)
~~~

##### 3.7.1.1.2、示例2：获得不同城市，不同年龄段用户的总消费、最长和最短页面驻留时间

1）创建rollup

~~~sql
alter table example_db.example_tbl_agg2 add rollup rollup_city_age_cost_maxdu_mindu(city,age,cost,max_dwell_time,min_dwell_time);
~~~

2）查看rollup的信息

~~~sql
mysql> SHOW ALTER TABLE ROLLUP;
+-------+------------------+---------------------+---------------------+------------------+----------------------------------+----------+---------------+----------+------+----------+---------+
| JobId | TableName        | CreateTime          | FinishTime          | BaseIndexName    | RollupIndexName                  | RollupId | TransactionId | State    | Msg  | Progress | Timeout |
+-------+------------------+---------------------+---------------------+------------------+----------------------------------+----------+---------------+----------+------+----------+---------+
| 12351 | example_tbl_agg2 | 2023-11-23 15:20:20 | 2023-11-23 15:20:21 | example_tbl_agg2 | rollup_user_cost                 | 12352    | 2073          | FINISHED |      | NULL     | 2592000 |
| 12355 | example_tbl_agg2 | 2023-11-23 15:29:52 | 2023-11-23 15:29:53 | example_tbl_agg2 | rollup_city_age_cost_maxdu_mindu | 12356    | 2074          | FINISHED |      | NULL     | 2592000 |
+-------+------------------+---------------------+---------------------+------------------+----------------------------------+----------+---------------+----------+------+----------+---------+
2 rows in set (0.00 sec)
~~~

3）通过explain命令查看sql是否使用到了rollup索引

~~~sql
explain SELECT city, age, sum(cost), max(max_dwell_time), min(min_dwell_time) FROM example_db.example_tbl_agg2 GROUP BY city, age;
mysql> explain SELECT city, age, sum(cost), max(max_dwell_time), min(min_dwell_time) FROM example_db.example_tbl_agg2 GROUP BY city, age;
+------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Explain String                                                                                                                                                   |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| PLAN FRAGMENT 0                                                                                                                                                  |
|   OUTPUT EXPRS:                                                                                                                                                  |
|     city[#25]                                                                                                                                                    |
|     age[#26]                                                                                                                                                     |
|     sum(cost)[#27]                                                                                                                                               |
|     max(max_dwell_time)[#28]                                                                                                                                     |
|     min(min_dwell_time)[#29]                                                                                                                                     |
|   PARTITION: UNPARTITIONED                                                                                                                                       |
|                                                                                                                                                                  |
|   VRESULT SINK                                                                                                                                                   |
|                                                                                                                                                                  |
|   4:VEXCHANGE                                                                                                                                                    |
|      offset: 0                                                                                                                                                   |
|                                                                                                                                                                  |
| PLAN FRAGMENT 1                                                                                                                                                  |
|                                                                                                                                                                  |
|   PARTITION: HASH_PARTITIONED: mv_city[#15], mv_age[#16]                                                                                                         |
|                                                                                                                                                                  |
|   STREAM DATA SINK                                                                                                                                               |
|     EXCHANGE ID: 04                                                                                                                                              |
|     UNPARTITIONED                                                                                                                                                |
|                                                                                                                                                                  |
|   3:VAGGREGATE (merge finalize)                                                                                                                                  |
|   |  output: sum(partial_sum(mva_SUM__cost)[#17])[#22], max(partial_max(mva_MAX__max_dwell_time)[#18])[#23], min(partial_min(mva_MIN__min_dwell_time)[#19])[#24] |
|   |  group by: mv_city[#15], mv_age[#16]                                                                                                                         |
|   |  cardinality=1                                                                                                                                               |
|   |  projections: mv_city[#20], mv_age[#21], sum(mva_SUM__cost)[#22], max(mva_MAX__max_dwell_time)[#23], min(mva_MIN__min_dwell_time)[#24]                       |
|   |  project output tuple id: 4                                                                                                                                  |
|   |                                                                                                                                                              |
|   2:VEXCHANGE                                                                                                                                                    |
|      offset: 0                                                                                                                                                   |
|                                                                                                                                                                  |
| PLAN FRAGMENT 2                                                                                                                                                  |
|                                                                                                                                                                  |
|   PARTITION: HASH_PARTITIONED: user_id[#5]                                                                                                                       |
|                                                                                                                                                                  |
|   STREAM DATA SINK                                                                                                                                               |
|     EXCHANGE ID: 02                                                                                                                                              |
|     HASH_PARTITIONED: mv_city[#15], mv_age[#16]                                                                                                                  |
|                                                                                                                                                                  |
|   1:VAGGREGATE (update serialize)                                                                                                                                |
|   |  STREAMING                                                                                                                                                   |
|   |  output: partial_sum(mva_SUM__cost[#2])[#17], partial_max(mva_MAX__max_dwell_time[#3])[#18], partial_min(mva_MIN__min_dwell_time[#4])[#19]                   |
|   |  group by: mv_city[#0], mv_age[#1]                                                                                                                           |
|   |  cardinality=3                                                                                                                                               |
|   |                                                                                                                                                              |
|   0:VOlapScanNode                                                                                                                                                |
|      TABLE: default_cluster:example_db.example_tbl_agg2(rollup_city_age_cost_maxdu_mindu), PREAGGREGATION: ON                                                    |
|      partitions=1/1, tablets=1/1, tabletList=12357                                                                                                               |
|      cardinality=7, avgRowSize=536.4286, numNodes=1                                                                                                              |
|      pushAggOp=NONE                                                                                                                                              |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------+
51 rows in set (0.03 sec)


explain SELECT city, sum(cost), max(max_dwell_time), min(min_dwell_time) FROM example_db.example_tbl_agg2 GROUP BY city;
mysql> explain SELECT city, sum(cost), max(max_dwell_time), min(min_dwell_time) FROM example_db.example_tbl_agg2 GROUP BY city;
+------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Explain String                                                                                                                                                   |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| PLAN FRAGMENT 0                                                                                                                                                  |
|   OUTPUT EXPRS:                                                                                                                                                  |
|     city[#23]                                                                                                                                                    |
|     sum(cost)[#24]                                                                                                                                               |
|     max(max_dwell_time)[#25]                                                                                                                                     |
|     min(min_dwell_time)[#26]                                                                                                                                     |
|   PARTITION: UNPARTITIONED                                                                                                                                       |
|                                                                                                                                                                  |
|   VRESULT SINK                                                                                                                                                   |
|                                                                                                                                                                  |
|   4:VEXCHANGE                                                                                                                                                    |
|      offset: 0                                                                                                                                                   |
|                                                                                                                                                                  |
| PLAN FRAGMENT 1                                                                                                                                                  |
|                                                                                                                                                                  |
|   PARTITION: HASH_PARTITIONED: mv_city[#15]                                                                                                                      |
|                                                                                                                                                                  |
|   STREAM DATA SINK                                                                                                                                               |
|     EXCHANGE ID: 04                                                                                                                                              |
|     UNPARTITIONED                                                                                                                                                |
|                                                                                                                                                                  |
|   3:VAGGREGATE (merge finalize)                                                                                                                                  |
|   |  output: sum(partial_sum(mva_SUM__cost)[#16])[#20], max(partial_max(mva_MAX__max_dwell_time)[#17])[#21], min(partial_min(mva_MIN__min_dwell_time)[#18])[#22] |
|   |  group by: mv_city[#15]                                                                                                                                      |
|   |  cardinality=1                                                                                                                                               |
|   |  projections: mv_city[#19], sum(mva_SUM__cost)[#20], max(mva_MAX__max_dwell_time)[#21], min(mva_MIN__min_dwell_time)[#22]                                    |
|   |  project output tuple id: 4                                                                                                                                  |
|   |                                                                                                                                                              |
|   2:VEXCHANGE                                                                                                                                                    |
|      offset: 0                                                                                                                                                   |
|                                                                                                                                                                  |
| PLAN FRAGMENT 2                                                                                                                                                  |
|                                                                                                                                                                  |
|   PARTITION: HASH_PARTITIONED: user_id[#5]                                                                                                                       |
|                                                                                                                                                                  |
|   STREAM DATA SINK                                                                                                                                               |
|     EXCHANGE ID: 02                                                                                                                                              |
|     HASH_PARTITIONED: mv_city[#15]                                                                                                                               |
|                                                                                                                                                                  |
|   1:VAGGREGATE (update serialize)                                                                                                                                |
|   |  STREAMING                                                                                                                                                   |
|   |  output: partial_sum(mva_SUM__cost[#2])[#16], partial_max(mva_MAX__max_dwell_time[#3])[#17], partial_min(mva_MIN__min_dwell_time[#4])[#18]                   |
|   |  group by: mv_city[#0]                                                                                                                                       |
|   |  cardinality=3                                                                                                                                               |
|   |                                                                                                                                                              |
|   0:VOlapScanNode                                                                                                                                                |
|      TABLE: default_cluster:example_db.example_tbl_agg2(rollup_city_age_cost_maxdu_mindu), PREAGGREGATION: ON                                                    |
|      partitions=1/1, tablets=1/1, tabletList=12357                                                                                                               |
|      cardinality=7, avgRowSize=536.4286, numNodes=1                                                                                                              |
|      pushAggOp=NONE                                                                                                                                              |
+------------------------------------------------------------------------------------------------------------------------------------------------------------------+
50 rows in set (0.03 sec)

    
explain SELECT city, age, sum(cost), min(min_dwell_time) FROM example_db.example_tbl_agg2 GROUP BY city, age;
mysql> explain SELECT city, age, sum(cost), min(min_dwell_time) FROM example_db.example_tbl_agg2 GROUP BY city, age;
+---------------------------------------------------------------------------------------------------------------+
| Explain String                                                                                                |
+---------------------------------------------------------------------------------------------------------------+
| PLAN FRAGMENT 0                                                                                               |
|   OUTPUT EXPRS:                                                                                               |
|     city[#23]                                                                                                 |
|     age[#24]                                                                                                  |
|     sum(cost)[#25]                                                                                            |
|     min(min_dwell_time)[#26]                                                                                  |
|   PARTITION: UNPARTITIONED                                                                                    |
|                                                                                                               |
|   VRESULT SINK                                                                                                |
|                                                                                                               |
|   4:VEXCHANGE                                                                                                 |
|      offset: 0                                                                                                |
|                                                                                                               |
| PLAN FRAGMENT 1                                                                                               |
|                                                                                                               |
|   PARTITION: HASH_PARTITIONED: mv_city[#15], mv_age[#16]                                                      |
|                                                                                                               |
|   STREAM DATA SINK                                                                                            |
|     EXCHANGE ID: 04                                                                                           |
|     UNPARTITIONED                                                                                             |
|                                                                                                               |
|   3:VAGGREGATE (merge finalize)                                                                               |
|   |  output: sum(partial_sum(mva_SUM__cost)[#17])[#21], min(partial_min(mva_MIN__min_dwell_time)[#18])[#22]   |
|   |  group by: mv_city[#15], mv_age[#16]                                                                      |
|   |  cardinality=1                                                                                            |
|   |  projections: mv_city[#19], mv_age[#20], sum(mva_SUM__cost)[#21], min(mva_MIN__min_dwell_time)[#22]       |
|   |  project output tuple id: 4                                                                               |
|   |                                                                                                           |
|   2:VEXCHANGE                                                                                                 |
|      offset: 0                                                                                                |
|                                                                                                               |
| PLAN FRAGMENT 2                                                                                               |
|                                                                                                               |
|   PARTITION: HASH_PARTITIONED: user_id[#5]                                                                    |
|                                                                                                               |
|   STREAM DATA SINK                                                                                            |
|     EXCHANGE ID: 02                                                                                           |
|     HASH_PARTITIONED: mv_city[#15], mv_age[#16]                                                               |
|                                                                                                               |
|   1:VAGGREGATE (update serialize)                                                                             |
|   |  STREAMING                                                                                                |
|   |  output: partial_sum(mva_SUM__cost[#2])[#17], partial_min(mva_MIN__min_dwell_time[#4])[#18]               |
|   |  group by: mv_city[#0], mv_age[#1]                                                                        |
|   |  cardinality=3                                                                                            |
|   |                                                                                                           |
|   0:VOlapScanNode                                                                                             |
|      TABLE: default_cluster:example_db.example_tbl_agg2(rollup_city_age_cost_maxdu_mindu), PREAGGREGATION: ON |
|      partitions=1/1, tablets=1/1, tabletList=12357                                                            |
|      cardinality=7, avgRowSize=536.4286, numNodes=1                                                           |
|      pushAggOp=NONE                                                                                           |
+---------------------------------------------------------------------------------------------------------------+
50 rows in set (0.02 sec)
~~~

#### 3.7.3、Duplicate 模型中的 ROLLUP

因为 Duplicate 模型没有聚合的语意。所以该模型中的 ROLLUP，已经失去了“上卷”这一层含义。而仅仅是作为调整列顺序，以命中前缀索引的作用。下面详细介绍前缀索引，以及如何使用ROLLUP改变前缀索引，以获得更好的查询效率。

##### 3.7.3.1、前缀索引

不同于传统的数据库设计，Doris 不支持在任意列上创建索引。Doris 这类 MPP 架构的 OLAP 数据库，通常都是通过提高并发，来处理大量数据的。

本质上，Doris 的数据存储在类似 SSTable（Sorted String Table）的数据结构中。该结构是一种有序的数据结构，可以按照指定的列进行排序存储。在这种数据结构上，以排序列作为条件进行查找，会非常的高效。

在 Aggregate、Unique 和 Duplicate 三种数据模型中。底层的数据存储，是按照各自建表语句中，AGGREGATE KEY、UNIQUE KEY 和 DUPLICATE KEY 中指定的列进行排序存储的。

而前缀索引，**即在排序的基础上，实现的一种根据给定前缀列，快速查询数据的索引方式**。

###### 3.7.3.1.1、**示例**

我们将一行数据的前 **36 个字节** 作为这行数据的前缀索引。当遇到 VARCHAR 类型时，前缀索引会直接截断。我们举例说明：

1. 以下表结构的前缀索引为 user_id(8 Bytes) + age(4 Bytes) + message(prefix 20 Bytes)。

    | ColumnName     | Type         |
    | -------------- | ------------ |
    | user_id        | BIGINT       |
    | age            | INT          |
    | message        | VARCHAR(100) |
    | max_dwell_time | DATETIME     |
    | min_dwell_time | DATETIME     |

2. 以下表结构的前缀索引为 user_name(20 Bytes)。即使没有达到 36 个字节，因为遇到 VARCHAR，所以直接截断，不再往后继续。

    | ColumnName     | Type         |
    | -------------- | ------------ |
    | user_name      | VARCHAR(20)  |
    | age            | INT          |
    | message        | VARCHAR(100) |
    | max_dwell_time | DATETIME     |
    | min_dwell_time | DATETIME     |

当我们的查询条件，是**前缀索引的前缀**时，可以极大的加快查询速度。比如在第一个例子中，我们执行如下查询：

```sql
SELECT * FROM table WHERE user_id=1829239 and age=20；
```

该查询的效率会**远高于**如下查询：

```sql
SELECT * FROM table WHERE age=20；
```

所以在建表时，**正确的选择列顺序，能够极大地提高查询效率**。

##### 3.7.3.2、ROLLUP 调整前缀索引

因为建表时已经指定了列顺序，所以一个表只有一种前缀索引。这对于使用其他不能命中前缀索引的列作为条件进行的查询来说，效率上可能无法满足需求。因此，我们可以通过创建 ROLLUP 来人为的调整列顺序。举例说明：

Base 表结构如下：

| ColumnName     | Type         |
| -------------- | ------------ |
| user_id        | BIGINT       |
| age            | INT          |
| message        | VARCHAR(100) |
| max_dwell_time | DATETIME     |
| min_dwell_time | DATETIME     |

我们可以在此基础上创建一个 ROLLUP 表：

| ColumnName     | Type         |
| -------------- | ------------ |
| age            | INT          |
| user_id        | BIGINT       |
| message        | VARCHAR(100) |
| max_dwell_time | DATETIME     |
| min_dwell_time | DATETIME     |

可以看到，ROLLUP 和 Base 表的列完全一样，只是将 user_id 和 age 的顺序调换了。那么当我们进行如下查询时：

```sql
mysql> SELECT * FROM table where age=20 and message LIKE "%error%";
```

会优先选择 ROLLUP 表，因为 ROLLUP 的前缀索引匹配度更高。

##### 3.7.3.3、ROLLUP使用说明

- ROLLUP 最根本的作用是提高某些查询的查询效率（无论是通过聚合来减少数据量，还是修改列顺序以匹配前缀索引）。因此 ROLLUP 的含义已经超出了 “上卷” 的范围。这也是为什么我们在源代码中，将其命名为 Materialized Index（物化索引）的原因。
- ROLLUP 是附属于 Base 表的，可以看做是 Base 表的一种辅助数据结构。用户可以在 Base 表的基础上，创建或删除 ROLLUP，但是不能在查询中显式的指定查询某 ROLLUP。是否命中 ROLLUP 完全由 Doris 系统自动决定。
- ROLLUP 的数据是独立物理存储的。因此，创建的 ROLLUP 越多，占用的磁盘空间也就越大。同时对导入速度也会有影响（导入的ETL阶段会自动产生所有 ROLLUP 的数据），但是不会降低查询效率（只会更好）。
- ROLLUP 的数据更新与 Base 表是完全同步的。用户无需关心这个问题。
- ROLLUP 中列的聚合方式，与 Base 表完全相同。在创建 ROLLUP 无需指定，也不能修改。
- 查询能否命中 ROLLUP 的一个必要条件（非充分条件）是，查询所涉及的**所有列**（包括 select list 和 where 中的查询条件列等）都存在于该 ROLLUP 的列中。否则，查询只能命中 Base 表。
- 某些类型的查询（如 count(*)）在任何条件下，都无法命中 ROLLUP。具体参见接下来的 **聚合模型的局限性** 一节。
- 可以通过 `EXPLAIN your_sql;` 命令获得查询执行计划，在执行计划中，查看是否命中 ROLLUP。
- 可以通过 `DESC tbl_name ALL;` 语句显示 Base 表和所有已创建完成的 ROLLUP。

#### 3.7.4、查询

在 Doris 里 Rollup 作为一份聚合物化视图，其在查询中可以起到两个作用：

- 索引
- 聚合数据（仅用于聚合模型，即aggregate key）

但是为了命中 Rollup 需要满足一定的条件，并且可以通过执行计划中 ScanNode 节点的 PreAggregation 的值来判断是否可以命中 Rollup，以及 Rollup 字段来判断命中的是哪一张 Rollup 表。

##### 3.7.4.1、索引

前面的3.7.3.1.小节中已经介绍过 Doris 的前缀索引，即 Doris 会把 Base/Rollup 表中的前 36 个字节（有 varchar 类型则可能导致前缀索引不满 36 个字节，varchar 会截断前缀索引，并且最多使用 varchar 的 20 个字节）在底层存储引擎单独生成一份排序的稀疏索引数据(数据也是排序的，用索引定位，然后在数据中做二分查找)，然后在查询的时候会根据查询中的条件来匹配每个 Base/Rollup 的前缀索引，并且选择出匹配前缀索引最长的一个 Base/Rollup。

```text
       -----> 从左到右匹配
+----+----+----+----+----+----+
| c1 | c2 | c3 | c4 | c5 |... |
```

如上图，取查询中 where 以及 on 上下推到 ScanNode 的条件，从前缀索引的第一列开始匹配，检查条件中是否有这些列，有则累计匹配的长度，直到匹配不上或者36字节结束（varchar类型的列只能匹配20个字节，并且会匹配不足36个字节截断前缀索引），然后选择出匹配长度最长的一个 Base/Rollup，下面举例说明，创建了一张Base表以及四张rollup：

```text
+---------------+-------+--------------+------+-------+---------+-------+
| IndexName     | Field | Type         | Null | Key   | Default | Extra |
+---------------+-------+--------------+------+-------+---------+-------+
| test          | k1    | TINYINT      | Yes  | true  | N/A     |       |
|               | k2    | SMALLINT     | Yes  | true  | N/A     |       |
|               | k3    | INT          | Yes  | true  | N/A     |       |
|               | k4    | BIGINT       | Yes  | true  | N/A     |       |
|               | k5    | DECIMAL(9,3) | Yes  | true  | N/A     |       |
|               | k6    | CHAR(5)      | Yes  | true  | N/A     |       |
|               | k7    | DATE         | Yes  | true  | N/A     |       |
|               | k8    | DATETIME     | Yes  | true  | N/A     |       |
|               | k9    | VARCHAR(20)  | Yes  | true  | N/A     |       |
|               | k10   | DOUBLE       | Yes  | false | N/A     | MAX   |
|               | k11   | FLOAT        | Yes  | false | N/A     | SUM   |
|               |       |              |      |       |         |       |
| rollup_index1 | k9    | VARCHAR(20)  | Yes  | true  | N/A     |       |
|               | k1    | TINYINT      | Yes  | true  | N/A     |       |
|               | k2    | SMALLINT     | Yes  | true  | N/A     |       |
|               | k3    | INT          | Yes  | true  | N/A     |       |
|               | k4    | BIGINT       | Yes  | true  | N/A     |       |
|               | k5    | DECIMAL(9,3) | Yes  | true  | N/A     |       |
|               | k6    | CHAR(5)      | Yes  | true  | N/A     |       |
|               | k7    | DATE         | Yes  | true  | N/A     |       |
|               | k8    | DATETIME     | Yes  | true  | N/A     |       |
|               | k10   | DOUBLE       | Yes  | false | N/A     | MAX   |
|               | k11   | FLOAT        | Yes  | false | N/A     | SUM   |
|               |       |              |      |       |         |       |
| rollup_index2 | k9    | VARCHAR(20)  | Yes  | true  | N/A     |       |
|               | k2    | SMALLINT     | Yes  | true  | N/A     |       |
|               | k1    | TINYINT      | Yes  | true  | N/A     |       |
|               | k3    | INT          | Yes  | true  | N/A     |       |
|               | k4    | BIGINT       | Yes  | true  | N/A     |       |
|               | k5    | DECIMAL(9,3) | Yes  | true  | N/A     |       |
|               | k6    | CHAR(5)      | Yes  | true  | N/A     |       |
|               | k7    | DATE         | Yes  | true  | N/A     |       |
|               | k8    | DATETIME     | Yes  | true  | N/A     |       |
|               | k10   | DOUBLE       | Yes  | false | N/A     | MAX   |
|               | k11   | FLOAT        | Yes  | false | N/A     | SUM   |
|               |       |              |      |       |         |       |
| rollup_index3 | k4    | BIGINT       | Yes  | true  | N/A     |       |
|               | k5    | DECIMAL(9,3) | Yes  | true  | N/A     |       |
|               | k6    | CHAR(5)      | Yes  | true  | N/A     |       |
|               | k1    | TINYINT      | Yes  | true  | N/A     |       |
|               | k2    | SMALLINT     | Yes  | true  | N/A     |       |
|               | k3    | INT          | Yes  | true  | N/A     |       |
|               | k7    | DATE         | Yes  | true  | N/A     |       |
|               | k8    | DATETIME     | Yes  | true  | N/A     |       |
|               | k9    | VARCHAR(20)  | Yes  | true  | N/A     |       |
|               | k10   | DOUBLE       | Yes  | false | N/A     | MAX   |
|               | k11   | FLOAT        | Yes  | false | N/A     | SUM   |
|               |       |              |      |       |         |       |
| rollup_index4 | k4    | BIGINT       | Yes  | true  | N/A     |       |
|               | k6    | CHAR(5)      | Yes  | true  | N/A     |       |
|               | k5    | DECIMAL(9,3) | Yes  | true  | N/A     |       |
|               | k1    | TINYINT      | Yes  | true  | N/A     |       |
|               | k2    | SMALLINT     | Yes  | true  | N/A     |       |
|               | k3    | INT          | Yes  | true  | N/A     |       |
|               | k7    | DATE         | Yes  | true  | N/A     |       |
|               | k8    | DATETIME     | Yes  | true  | N/A     |       |
|               | k9    | VARCHAR(20)  | Yes  | true  | N/A     |       |
|               | k10   | DOUBLE       | Yes  | false | N/A     | MAX   |
|               | k11   | FLOAT        | Yes  | false | N/A     | SUM   |
+---------------+-------+--------------+------+-------+---------+-------+
```

这五张表的前缀索引分别为

```text
Base(k1 ,k2, k3, k4, k5, k6, k7)

rollup_index1(k9)

rollup_index2(k9)

rollup_index3(k4, k5, k6, k1, k2, k3, k7)

rollup_index4(k4, k6, k5, k1, k2, k3, k7)
```

能用的上前缀索引的列上的条件需要是 `=` `<` `>` `<=` `>=` `in` `between` 这些并且这些条件是并列的且关系使用 `and` 连接，对于`or`、`!=` 等这些不能命中，然后看以下查询：

```sql
SELECT * FROM test WHERE k1 = 1 AND k2 > 3;
```

有 k1 以及 k2 上的条件，检查只有 Base 的第一列含有条件里的 k1，所以匹配最长的前缀索引即 test，explain一下：

```sql
|   0:OlapScanNode
|      TABLE: test
|      PREAGGREGATION: OFF. Reason: No AggregateInfo
|      PREDICATES: `k1` = 1, `k2` > 3
|      partitions=1/1
|      rollup: test
|      buckets=1/10
|      cardinality=-1
|      avgRowSize=0.0
|      numNodes=0
|      tuple ids: 0
```

再看以下查询：

```sql
SELECT * FROM test WHERE k4 = 1 AND k5 > 3;
```

有 k4 以及 k5 的条件，检查 rollup_index3、rollup_index4 的第一列含有 k4，但是 rollup_index3 的第二列含有k5，所以匹配的前缀索引最长。

```sql
|   0:OlapScanNode
|      TABLE: test
|      PREAGGREGATION: OFF. Reason: No AggregateInfo
|      PREDICATES: `k4` = 1, `k5` > 3
|      partitions=1/1
|      rollup: rollup_index3
|      buckets=10/10
|      cardinality=-1
|      avgRowSize=0.0
|      numNodes=0
|      tuple ids: 0
```

现在我们尝试匹配含有 varchar 列上的条件，如下：

```sql
SELECT * FROM test WHERE k9 IN ("xxx", "yyyy") AND k1 = 10;
```

有 k9 以及 k1 两个条件，rollup_index1 以及 rollup_index2 的第一列都含有 k9，按理说这里选择这两个 rollup 都可以命中前缀索引并且效果是一样的随机选择一个即可（因为这里 varchar 刚好20个字节，前缀索引不足36个字节被截断），但是当前策略这里还会继续匹配 k1，因为 rollup_index1 的第二列为 k1，所以选择了 rollup_index1，其实后面的 k1 条件并不会起到加速的作用。(如果对于前缀索引外的条件需要其可以起到加速查询的目的，可以通过建立 Bloom Filter 过滤器加速。一般对于字符串类型建立即可，因为 Doris 针对列存在 Block 级别对于整型、日期已经有 Min/Max 索引) 以下是 explain 的结果。

```text
|   0:OlapScanNode
|      TABLE: test
|      PREAGGREGATION: OFF. Reason: No AggregateInfo
|      PREDICATES: `k9` IN ('xxx', 'yyyy'), `k1` = 10
|      partitions=1/1
|      rollup: rollup_index1
|      buckets=1/10
|      cardinality=-1
|      avgRowSize=0.0
|      numNodes=0
|      tuple ids: 0
```

最后看一个多张Rollup都可以命中的查询：

```sql
SELECT * FROM test WHERE k4 < 1000 AND k5 = 80 AND k6 >= 10000;
```

有 k4,k5,k6 三个条件，rollup_index3 以及 rollup_index4 的前3列分别含有这三列，所以两者匹配的前缀索引长度一致，选取两者都可以，当前默认的策略为选取了比较早创建的一张 rollup，这里为 rollup_index3。

```text
|   0:OlapScanNode
|      TABLE: test
|      PREAGGREGATION: OFF. Reason: No AggregateInfo
|      PREDICATES: `k4` < 1000, `k5` = 80, `k6` >= 10000.0
|      partitions=1/1
|      rollup: rollup_index3
|      buckets=10/10
|      cardinality=-1
|      avgRowSize=0.0
|      numNodes=0
|      tuple ids: 0
```

如果稍微修改上面的查询为：

```sql
SELECT * FROM test WHERE k4 < 1000 AND k5 = 80 OR k6 >= 10000;
```

则这里的查询不能命中前缀索引。（甚至 Doris 存储引擎内的任何 Min/Max,BloomFilter 索引都不能起作用)

##### 3.7.4.2、聚合数据

当然一般的聚合物化视图其聚合数据的功能是必不可少的，这类物化视图对于聚合类查询或报表类查询都有非常大的帮助，要命中聚合物化视图需要下面一些前提：

1. 查询或者子查询中涉及的所有列都存在一张独立的 Rollup 中。
2. 如果查询或者子查询中有 Join，则 Join 的类型需要是 Inner join。

以下是可以命中Rollup的一些聚合查询的种类，

| 列类型 查询类型 | Sum   | Distinct/Count Distinct | Min   | Max   | APPROX_COUNT_DISTINCT |
| --------------- | ----- | ----------------------- | ----- | ----- | --------------------- |
| Key             | false | true                    | true  | true  | true                  |
| Value(Sum)      | true  | false                   | false | false | false                 |
| Value(Replace)  | false | false                   | false | false | false                 |
| Value(Min)      | false | false                   | true  | false | false                 |
| Value(Max)      | false | false                   | false | true  | false                 |

如果符合上述条件，则针对聚合模型在判断命中 Rollup 的时候会有两个阶段：

1. 首先通过条件匹配出命中前缀索引索引最长的 Rollup 表，见上述索引策略。
2. 然后比较 Rollup 的行数，选择最小的一张 Rollup。

如下 Base 表以及 Rollup：

```sql
+-------------+-------+--------------+------+-------+---------+-------+
| IndexName   | Field | Type         | Null | Key   | Default | Extra |
+-------------+-------+--------------+------+-------+---------+-------+
| test_rollup | k1    | TINYINT      | Yes  | true  | N/A     |       |
|             | k2    | SMALLINT     | Yes  | true  | N/A     |       |
|             | k3    | INT          | Yes  | true  | N/A     |       |
|             | k4    | BIGINT       | Yes  | true  | N/A     |       |
|             | k5    | DECIMAL(9,3) | Yes  | true  | N/A     |       |
|             | k6    | CHAR(5)      | Yes  | true  | N/A     |       |
|             | k7    | DATE         | Yes  | true  | N/A     |       |
|             | k8    | DATETIME     | Yes  | true  | N/A     |       |
|             | k9    | VARCHAR(20)  | Yes  | true  | N/A     |       |
|             | k10   | DOUBLE       | Yes  | false | N/A     | MAX   |
|             | k11   | FLOAT        | Yes  | false | N/A     | SUM   |
|             |       |              |      |       |         |       |
| rollup2     | k1    | TINYINT      | Yes  | true  | N/A     |       |
|             | k2    | SMALLINT     | Yes  | true  | N/A     |       |
|             | k3    | INT          | Yes  | true  | N/A     |       |
|             | k10   | DOUBLE       | Yes  | false | N/A     | MAX   |
|             | k11   | FLOAT        | Yes  | false | N/A     | SUM   |
|             |       |              |      |       |         |       |
| rollup1     | k1    | TINYINT      | Yes  | true  | N/A     |       |
|             | k2    | SMALLINT     | Yes  | true  | N/A     |       |
|             | k3    | INT          | Yes  | true  | N/A     |       |
|             | k4    | BIGINT       | Yes  | true  | N/A     |       |
|             | k5    | DECIMAL(9,3) | Yes  | true  | N/A     |       |
|             | k10   | DOUBLE       | Yes  | false | N/A     | MAX   |
|             | k11   | FLOAT        | Yes  | false | N/A     | SUM   |
+-------------+-------+--------------+------+-------+---------+-------+
```

看以下查询：

```sql
SELECT SUM(k11) FROM test_rollup WHERE k1 = 10 AND k2 > 200 AND k3 in (1,2,3);
```

首先判断查询是否可以命中聚合的 Rollup表，经过查上面的图是可以的，然后条件中含有 k1,k2,k3 三个条件，这三个条件 test_rollup、rollup1、rollup2 的前三列都含有，所以前缀索引长度一致，然后比较行数显然 rollup2 的聚合程度最高行数最少所以选取 rollup2。

```sql
|   0:OlapScanNode                                          |
|      TABLE: test_rollup                                   |
|      PREAGGREGATION: ON                                   |
|      PREDICATES: `k1` = 10, `k2` > 200, `k3` IN (1, 2, 3) |
|      partitions=1/1                                       |
|      rollup: rollup2                                      |
|      buckets=1/10                                         |
|      cardinality=-1                                       |
|      avgRowSize=0.0                                       |
|      numNodes=0                                           |
|      tuple ids: 0                                         |
```

### 3.8、物化视图

物化视图是将预先计算（根据定义好的 SELECT 语句）好的数据集，存储在 Doris 中的一个特殊的表。

物化视图的出现主要是为了满足用户，既能对原始明细数据的任意维度分析，也能快速的对固定维度进行分析查询。

#### 3.8.1、适用场景

- 分析需求覆盖明细数据查询以及固定维度查询两方面。
- 查询仅涉及表中的很小一部分列或行。
- 查询包含一些耗时处理操作，比如：时间很久的聚合操作等。
- 查询需要匹配不同前缀索引。

#### 3.8.2、优势

- 对于那些经常重复的使用相同的子查询结果的查询性能大幅提升。
- Doris 自动维护物化视图的数据，无论是新的导入，还是删除操作都能保证 Base 表和物化视图表的数据一致性，无需任何额外的人工维护成本。
- 查询时，会自动匹配到最优物化视图，并直接从物化视图中读取数据。

*自动维护物化视图的数据会造成一些维护开销，会在后面的物化视图的局限性中展开说明。*

#### 3.8.3、物化视图 VS Rollup

在没有物化视图功能之前，用户一般都是使用 Rollup 功能通过预聚合方式提升查询效率的。但是 Rollup 具有一定的局限性，他不能基于明细模型做预聚合。

物化视图则在覆盖了 Rollup 的功能的同时，还能支持更丰富的聚合函数。所以物化视图其实是 Rollup 的一个超集。

也就是说，之前 [ALTER TABLE ADD ROLLUP](https://doris.apache.org/zh-CN/docs/sql-manual/sql-reference/Data-Definition-Statements/Alter/ALTER-TABLE-ROLLUP) 语法支持的功能现在均可以通过 [CREATE MATERIALIZED VIEW](https://doris.apache.org/zh-CN/docs/sql-manual/sql-reference/Data-Definition-Statements/Create/CREATE-MATERIALIZED-VIEW) 实现。

#### 3.8.4、使用物化视图

Doris 系统提供了一整套对物化视图的 DDL 语法，包括创建，查看，删除。DDL 的语法和 PostgreSQL, Oracle 都是一致的。

##### 3.8.4.1、创建物化视图

这里首先你要根据你的查询语句的特点来决定创建一个什么样的物化视图。这里并不是说你的物化视图定义和你的某个查询语句一模一样就最好。这里有两个原则：

1. 从查询语句中**抽象**出，多个查询共有的分组和聚合方式作为物化视图的定义。
2. 不需要给所有维度组合都创建物化视图。

首先第一个点，一个物化视图如果抽象出来，并且多个查询都可以匹配到这张物化视图。这种物化视图效果最好。因为物化视图的维护本身也需要消耗资源。

如果物化视图只和某个特殊的查询很贴合，而其他查询均用不到这个物化视图。则会导致这张物化视图的性价比不高，既占用了集群的存储资源，还不能为更多的查询服务。

所以用户需要结合自己的查询语句，以及数据维度信息去抽象出一些物化视图的定义。

第二点就是，在实际的分析查询中，并不会覆盖到所有的维度分析。所以给常用的维度组合创建物化视图即可，从而到达一个空间和时间上的平衡。

创建物化视图是一个异步的操作，也就是说用户成功提交创建任务后，Doris 会在后台对存量的数据进行计算，直到创建成功。

具体的语法可查看[CREATE MATERIALIZED VIEW](https://doris.apache.org/zh-CN/docs/sql-manual/sql-reference/Data-Definition-Statements/Create/CREATE-MATERIALIZED-VIEW) 。

语法：

```sql
CREATE MATERIALIZED VIEW < MV name > as < query >
[PROPERTIES ("key" = "value")]
```

说明：

- `MV name`：物化视图的名称，必填项。相同表的物化视图名称不可重复。

- `query`：用于构建物化视图的查询语句，查询语句的结果既物化视图的数据。目前支持的 query 格式为:

    ```sql
    SELECT select_expr[, select_expr ...]
    FROM [Base view name]
    GROUP BY column_name[, column_name ...]
    ORDER BY column_name[, column_name ...]
    ```

    语法和查询语句语法一致。

    - ```sql
        select_expr
        ```

        ：物化视图的 schema 中所有的列。

        - 至少包含一个单列。

    - ```sql
        base view name
        ```

        ：物化视图的原始表名，必填项。

        - 必须是单表，且非子查询

    - ```sql
        group by
        ```

        ：物化视图的分组列，选填项。

        - 不填则数据不进行分组。

    - ```sql
        order by
        ```

        ：物化视图的排序列，选填项。

        - 排序列的声明顺序必须和 select_expr 中列声明顺序一致。
        - 如果不声明 order by，则根据规则自动补充排序列。 如果物化视图是聚合类型，则所有的分组列自动补充为排序列。 如果物化视图是非聚合类型，则前 36 个字节自动补充为排序列。
        - 如果自动补充的排序个数小于3个，则前三个作为排序列。 如果 query 中包含分组列的话，则排序列必须和分组列一致。

- properties

    声明物化视图的一些配置，选填项。

    ```sql
    PROPERTIES ("key" = "value", "key" = "value" ...)
    ```

    以下几个配置，均可声明在此处：

    ```text
     short_key: 排序列的个数。
     timeout: 物化视图构建的超时时间。
    ```

在`Doris 2.0`版本中我们对物化视图的做了一些增强(在本文的`最佳实践4`中有具体描述)。我们建议用户在正式的生产环境中使用物化视图前，先在测试环境中确认是预期中的查询能否命中想要创建的物化视图。

如果不清楚如何验证一个查询是否命中物化视图，可以阅读本文的`最佳实践1`。

与此同时，我们不建议用户在同一张表上建多个形态类似的物化视图，这可能会导致多个物化视图之间的冲突使得查询命中失败(在新优化器中这个问题会有所改善)。建议用户先在测试环境中验证物化视图和查询是否满足需求并能正常使用。

##### 3.8.4.2、支持聚合函数

目前物化视图创建语句支持的聚合函数有：

- SUM, MIN, MAX (Version 0.12)
- COUNT, BITMAP_UNION, HLL_UNION (Version 0.13)

##### 3.8.4.3、更新策略

为保证物化视图表和 Base 表的数据一致性, Doris 会将导入，删除等对 Base 表的操作都同步到物化视图表中。并且通过增量更新的方式来提升更新效率。通过事务方式来保证原子性。

比如如果用户通过 INSERT 命令插入数据到 Base 表中，则这条数据会同步插入到物化视图中。当 Base 表和物化视图表均写入成功后，INSERT 命令才会成功返回。

##### 3.8.4.4、查询自动匹配

物化视图创建成功后，用户的查询不需要发生任何改变，也就是还是查询的 Base 表。Doris 会根据当前查询的语句去自动选择一个最优的物化视图，从物化视图中读取数据并计算。

用户可以通过 EXPLAIN 命令来检查当前查询是否使用了物化视图。

物化视图中的聚合和查询中聚合的匹配关系：

| 物化视图聚合 | 查询中聚合                                             |
| ------------ | ------------------------------------------------------ |
| sum          | sum                                                    |
| min          | min                                                    |
| max          | max                                                    |
| count        | count                                                  |
| bitmap_union | bitmap_union, bitmap_union_count, count(distinct)      |
| hll_union    | hll_raw_agg, hll_union_agg, ndv, approx_count_distinct |

其中 bitmap 和 hll 的聚合函数在查询匹配到物化视图后，查询的聚合算子会根据物化视图的表结构进行改写。详细见实例2。

##### 3.8.4.5、查询物化视图

查看当前表都有哪些物化视图，以及他们的表结构都是什么样的。通过下面命令：

~~~sql
 desc table_name all;
~~~

##### 3.8.4.6、删除物化视图

如果用户不再需要物化视图，则可以通过命令删除物化视图。

具体的语法可查看[DROP MATERIALIZED VIEW](https://doris.apache.org/zh-CN/docs/sql-manual/sql-reference/Data-Definition-Statements/Drop/DROP-MATERIALIZED-VIEW)

语法：

```sql
DROP MATERIALIZED VIEW [IF EXISTS] mv_name ON table_name;
```

1. IF EXISTS: 如果物化视图不存在，不要抛出错误。如果不声明此关键字，物化视图不存在则报错。
2. mv_name: 待删除的物化视图的名称。必填项。
3. table_name: 待删除的物化视图所属的表名。必填项

##### 3.8.4.7、查看已创建的物化视图

用户可以通过命令查看已创建的物化视图的

具体的语法可查看[SHOW CREATE MATERIALIZED VIEW](https://doris.apache.org/zh-CN/docs/sql-manual/sql-reference/Show-Statements/SHOW-CREATE-MATERIALIZED-VIEW)

语法：

```sql
SHOW CREATE MATERIALIZED VIEW mv_name ON table_name
```

1. mv_name: 物化视图的名称。必填项。
2. table_name: 物化视图所属的表名。必填项。

##### 3.8.4.8、取消创建物化视图

```sql
 CANCEL ALTER TABLE MATERIALIZED VIEW FROM db_name.table_name
```

##### 3.8.4.9、查看创建物化视图提交的作业执行情况

> 该语句等同于 `SHOW ALTER TABLE ROLLUP`;

```sql
SHOW ALTER TABLE MATERIALIZED VIEW
[FROM database]
[WHERE]
[ORDER BY]
[LIMIT OFFSET]
```

- database：查看指定数据库下的作业。如不指定，使用当前数据库。
- WHERE：可以对结果列进行筛选，目前仅支持对以下列进行筛选：
    - TableName：仅支持等值筛选。
    - State：仅支持等值筛选。
    - Createtime/FinishTime：支持 =，>=，<=，>，<，!=
- ORDER BY：可以对结果集按任意列进行排序。
- LIMIT：配合 ORDER BY 进行翻页查询。

返回结果说明：

```sql
mysql> show alter table materialized view\G
*************************** 1. row ***************************
          JobId: 11001
      TableName: tbl1
     CreateTime: 2020-12-23 10:41:00
     FinishTime: NULL
  BaseIndexName: tbl1
RollupIndexName: r1
       RollupId: 11002
  TransactionId: 5070
          State: WAITING_TXN
            Msg:
       Progress: NULL
        Timeout: 86400
1 row in set (0.00 sec)
```

- `JobId`：作业唯一ID。

- `TableName`：基表名称

- `CreateTime/FinishTime`：作业创建时间和结束时间。

- `BaseIndexName/RollupIndexName`：基表名称和物化视图名称。

- `RollupId`：物化视图的唯一 ID。

- `TransactionId`：见 State 字段说明。

- `State`：作业状态。

    - PENDING：作业准备中。

    - WAITING_TXN：

        在正式开始产生物化视图数据前，会等待当前这个表上的正在运行的导入事务完成。而 `TransactionId` 字段就是当前正在等待的事务ID。当这个ID之前的导入都完成后，就会实际开始作业。

    - RUNNING：作业运行中。

    - FINISHED：作业运行成功。

    - CANCELLED：作业运行失败。

- `Msg`：错误信息

- `Progress`：作业进度。这里的进度表示 `已完成的tablet数量/总tablet数量`。创建物化视图是按 tablet 粒度进行的。

- `Timeout`：作业超时时间，单位秒。

#### 3.8.5、示例

##### 3.8.5.1、示例1：初步熟悉物化视图的使用过程

**第一步：创建物化视图**

假设用户有一张销售记录明细表，存储了每个交易的交易 id，销售员，售卖门店，销售时间，以及金额。建表语句和插入数据语句为：

~~~sql
create table sales_records(record_id int, seller_id int, store_id int, sale_date date, sale_amt bigint) distributed by hash(record_id) properties("replication_num" = "1");
insert into sales_records values(1,1,1,"2020-02-02",1);
~~~

这张 `sales_records` 的表结构如下：

~~~sql
mysql> 
mysql> desc sales_records;
+-----------+--------+------+-------+---------+-------+
| Field     | Type   | Null | Key   | Default | Extra |
+-----------+--------+------+-------+---------+-------+
| record_id | INT    | Yes  | true  | NULL    |       |
| seller_id | INT    | Yes  | true  | NULL    |       |
| store_id  | INT    | Yes  | true  | NULL    |       |
| sale_date | DATE   | Yes  | false | NULL    | NONE  |
| sale_amt  | BIGINT | Yes  | false | NULL    | NONE  |
+-----------+--------+------+-------+---------+-------+
5 rows in set (0.00 sec)
~~~

这时候如果用户经常对不同门店的销售量进行一个分析查询，则可以给这个 `sales_records` 表创建一张以售卖门店分组，对相同售卖门店的销售额求和的一个物化视图。创建语句如下：

~~~sql
create materialized view store_amt as select store_id, sum(sale_amt) from sales_records group by store_id;
~~~

后端返回下图，则说明创建物化视图任务提交成功。

~~~sql
mysql> create materialized view store_amt as select store_id, sum(sale_amt) from sales_records group by store_id;
Query OK, 0 rows affected (0.01 sec)
~~~

**第二步：检查物化视图是否构建完成**

由于创建物化视图是一个异步的操作，用户在提交完创建物化视图任务后，需要异步的通过命令检查物化视图是否构建完成。命令如下：

~~~sql
SHOW ALTER TABLE ROLLUP FROM sales_records; (Version 0.12)
SHOW ALTER TABLE MATERIALIZED VIEW FROM sales_records; (Version 0.13)
~~~

结果如下：

~~~sql
mysql> SHOW ALTER TABLE MATERIALIZED VIEW FROM example_db;
+-------+------------------+---------------------+---------------------+------------------+----------------------------------+----------+---------------+----------+------+----------+---------+
| JobId | TableName        | CreateTime          | FinishTime          | BaseIndexName    | RollupIndexName                  | RollupId | TransactionId | State    | Msg  | Progress | Timeout |
+-------+------------------+---------------------+---------------------+------------------+----------------------------------+----------+---------------+----------+------+----------+---------+
| 12351 | example_tbl_agg2 | 2023-11-23 15:20:20 | 2023-11-23 15:20:21 | example_tbl_agg2 | rollup_user_cost                 | 12352    | 2073          | FINISHED |      | NULL     | 2592000 |
| 12355 | example_tbl_agg2 | 2023-11-23 15:29:52 | 2023-11-23 15:29:53 | example_tbl_agg2 | rollup_city_age_cost_maxdu_mindu | 12356    | 2074          | FINISHED |      | NULL     | 2592000 |
| 12386 | sales_records    | 2023-11-23 16:50:25 | 2023-11-23 16:50:26 | sales_records    | store_amt                        | 12387    | 2082          | FINISHED |      | NULL     | 2592000 |
+-------+------------------+---------------------+---------------------+------------------+----------------------------------+----------+---------------+----------+------+----------+---------+
3 rows in set (0.00 sec)
~~~

其中 TableName 指的是物化视图的数据来自于哪个表，RollupIndexName 指的是物化视图的名称叫什么。其中比较重要的指标是 State。

当创建物化视图任务的 State 已经变成 FINISHED 后，就说明这个物化视图已经创建成功了。这就意味着，查询的时候有可能自动匹配到这张物化视图了。

**第三步：查询**

当创建完成物化视图后，用户再查询不同门店的销售量时，就会直接从刚才创建的物化视图 `store_amt` 中读取聚合好的数据。达到提升查询效率的效果。

用户的查询依旧指定查询 `sales_records` 表，比如：

```sql
SELECT store_id, sum(sale_amt) FROM sales_records GROUP BY store_id;
```

上面查询就能自动匹配到 `store_amt`。用户可以通过下面命令，检验当前查询是否匹配到了合适的物化视图。

~~~sql
mysql> explain SELECT store_id, sum(sale_amt) FROM sales_records GROUP BY store_id;
+-------------------------------------------------------------------------------------+
| Explain String                                                                      |
+-------------------------------------------------------------------------------------+
| PLAN FRAGMENT 0                                                                     |
|   OUTPUT EXPRS:                                                                     |
|     store_id[#11]                                                                   |
|     sum(sale_amt)[#12]                                                              |
|   PARTITION: UNPARTITIONED                                                          |
|                                                                                     |
|   VRESULT SINK                                                                      |
|                                                                                     |
|   4:VEXCHANGE                                                                       |
|      offset: 0                                                                      |
|                                                                                     |
| PLAN FRAGMENT 1                                                                     |
|                                                                                     |
|   PARTITION: HASH_PARTITIONED: mv_store_id[#7]                                      |
|                                                                                     |
|   STREAM DATA SINK                                                                  |
|     EXCHANGE ID: 04                                                                 |
|     UNPARTITIONED                                                                   |
|                                                                                     |
|   3:VAGGREGATE (merge finalize)                                                     |
|   |  output: sum(partial_sum(mva_SUM__sale_amt)[#8])[#10]                           |
|   |  group by: mv_store_id[#7]                                                      |
|   |  cardinality=1                                                                  |
|   |  projections: mv_store_id[#9], sum(mva_SUM__sale_amt)[#10]                      |
|   |  project output tuple id: 4                                                     |
|   |                                                                                 |
|   2:VEXCHANGE                                                                       |
|      offset: 0                                                                      |
|                                                                                     |
| PLAN FRAGMENT 2                                                                     |
|                                                                                     |
|   PARTITION: HASH_PARTITIONED: record_id[#2]                                        |
|                                                                                     |
|   STREAM DATA SINK                                                                  |
|     EXCHANGE ID: 02                                                                 |
|     HASH_PARTITIONED: mv_store_id[#7]                                               |
|                                                                                     |
|   1:VAGGREGATE (update serialize)                                                   |
|   |  STREAMING                                                                      |
|   |  output: partial_sum(mva_SUM__sale_amt[#1])[#8]                                 |
|   |  group by: mv_store_id[#0]                                                      |
|   |  cardinality=1                                                                  |
|   |                                                                                 |
|   0:VOlapScanNode                                                                   |
|      TABLE: default_cluster:example_db.sales_records(store_amt), PREAGGREGATION: ON |
|      partitions=1/1, tablets=10/10, tabletList=12388,12390,12392 ...                |
|      cardinality=1, avgRowSize=1520.0, numNodes=3                                   |
|      pushAggOp=NONE                                                                 |
+-------------------------------------------------------------------------------------+
48 rows in set (0.04 sec)
~~~

从最底部的`example_db.sales_records(store_amt)`可以表明这个查询命中了`store_amt`这个物化视图。值得注意的是，如果表中没有数据，那么可能不会命中物化视图。

##### 3.8.5.2、示例2：计算广告的 UV，PV

假设用户的原始广告点击数据存储在 Doris，那么针对广告 PV, UV 查询就可以通过创建 `bitmap_union` 的物化视图来提升查询速度。

**1、建表**

~~~sql
create table advertiser_view_record(time date, advertiser varchar(10), channel varchar(10), user_id int) distributed by hash(time) properties("replication_num" = "1");
insert into advertiser_view_record values("2020-02-02",'a','a',1);
~~~

**2、创建物化视图**

由于用户想要查询的是广告的 UV 值，也就是需要对相同广告的用户进行一个精确去重，则查询一般为：

```sql
SELECT advertiser, channel, count(distinct user_id) FROM advertiser_view_record GROUP BY advertiser, channel;
```

针对这种求 UV 的场景，我们就可以创建一个带 `bitmap_union` 的物化视图从而达到一个预先精确去重的效果。

在 Doris 中，`count(distinct)` 聚合的结果和 `bitmap_union_count`聚合的结果是完全一致的。而`bitmap_union_count` 等于 `bitmap_union` 的结果求 count， 所以如果查询中**涉及到 `count(distinct)` 则通过创建带 `bitmap_union` 聚合的物化视图方可加快查询**。

针对这个 Case，则可以创建一个根据广告和渠道分组，对 `user_id` 进行精确去重的物化视图。

```sql
create materialized view advertiser_uv as select advertiser, channel, bitmap_union(to_bitmap(user_id)) from advertiser_view_record group by advertiser, channel;
```

*注意：因为本身 user_id 是一个 INT 类型，所以在 Doris 中需要先将字段通过函数 `to_bitmap` 转换为 bitmap 类型然后才可以进行 `bitmap_union` 聚合。*

**3、查询自动匹配**

当物化视图表创建完成后，查询广告 UV 时，Doris 就会自动从刚才创建好的物化视图 `advertiser_uv` 中查询数据。比如原始的查询语句如下：

```sql
SELECT advertiser, channel, count(distinct user_id) FROM advertiser_view_record GROUP BY advertiser, channel;
```

在选中物化视图后，实际的查询会转化为：

```sql
SELECT advertiser, channel, bitmap_union_count(to_bitmap(user_id)) FROM advertiser_uv GROUP BY advertiser, channel;
```

通过 EXPLAIN 命令可以检验到 Doris 是否匹配到了物化视图：

~~~sql
mysql> explain SELECT advertiser, channel, count(distinct user_id) FROM advertiser_view_record GROUP BY advertiser, channel;
+-------------------------------------------------------------------------------------------------------------------------------------------------+
| Explain String                                                                                                                                  |
+-------------------------------------------------------------------------------------------------------------------------------------------------+
| PLAN FRAGMENT 0                                                                                                                                 |
|   OUTPUT EXPRS:                                                                                                                                 |
|     advertiser[#13]                                                                                                                             |
|     channel[#14]                                                                                                                                |
|     count(DISTINCT user_id)[#15]                                                                                                                |
|   PARTITION: UNPARTITIONED                                                                                                                      |
|                                                                                                                                                 |
|   VRESULT SINK                                                                                                                                  |
|                                                                                                                                                 |
|   4:VEXCHANGE                                                                                                                                   |
|      offset: 0                                                                                                                                  |
|                                                                                                                                                 |
| PLAN FRAGMENT 1                                                                                                                                 |
|                                                                                                                                                 |
|   PARTITION: HASH_PARTITIONED: mv_advertiser[#7], mv_channel[#8]                                                                                |
|                                                                                                                                                 |
|   STREAM DATA SINK                                                                                                                              |
|     EXCHANGE ID: 04                                                                                                                             |
|     UNPARTITIONED                                                                                                                               |
|                                                                                                                                                 |
|   3:VAGGREGATE (merge finalize)                                                                                                                 |
|   |  output: bitmap_union_count(partial_bitmap_union_count(mva_BITMAP_UNION__to_bitmap_with_check(cast(user_id as BIGINT)))[#9])[#12]           |
|   |  group by: mv_advertiser[#7], mv_channel[#8]                                                                                                |
|   |  cardinality=1                                                                                                                              |
|   |  projections: mv_advertiser[#10], mv_channel[#11], bitmap_union_count(mva_BITMAP_UNION__to_bitmap_with_check(cast(user_id as BIGINT)))[#12] |
|   |  project output tuple id: 4                                                                                                                 |
|   |                                                                                                                                             |
|   2:VEXCHANGE                                                                                                                                   |
|      offset: 0                                                                                                                                  |
|                                                                                                                                                 |
| PLAN FRAGMENT 2                                                                                                                                 |
|                                                                                                                                                 |
|   PARTITION: HASH_PARTITIONED: time[#3]                                                                                                         |
|                                                                                                                                                 |
|   STREAM DATA SINK                                                                                                                              |
|     EXCHANGE ID: 02                                                                                                                             |
|     HASH_PARTITIONED: mv_advertiser[#7], mv_channel[#8]                                                                                         |
|                                                                                                                                                 |
|   1:VAGGREGATE (update serialize)                                                                                                               |
|   |  STREAMING                                                                                                                                  |
|   |  output: partial_bitmap_union_count(mva_BITMAP_UNION__to_bitmap_with_check(cast(user_id as BIGINT))[#2])[#9]                                |
|   |  group by: mv_advertiser[#0], mv_channel[#1]                                                                                                |
|   |  cardinality=1                                                                                                                              |
|   |                                                                                                                                             |
|   0:VOlapScanNode                                                                                                                               |
|      TABLE: default_cluster:example_db.advertiser_view_record(advertiser_uv), PREAGGREGATION: ON                                                |
|      partitions=1/1, tablets=10/10, tabletList=12436,12438,12440 ...                                                                            |
|      cardinality=1, avgRowSize=2610.0, numNodes=3                                                                                               |
|      pushAggOp=NONE                                                                                                                             |
+-------------------------------------------------------------------------------------------------------------------------------------------------+
49 rows in set (0.05 sec)
~~~

在 EXPLAIN 的结果中，首先可以看到 `VOlapScanNode` 命中了 `advertiser_uv`。也就是说，查询会直接扫描物化视图的数据。说明匹配成功。

其次对于 `user_id` 字段求 `count(distinct)` 被改写为求 `bitmap_union_count(to_bitmap)`。也就是通过 Bitmap 的方式来达到精确去重的效果。

##### 3.8.5.3、示例3：匹配更丰富的前缀索引

用户的原始表有 （k1, k2, k3） 三列。其中 k1, k2 为前缀索引列。这时候如果用户查询条件中包含 `where k1=1 and k2=2` 就能通过索引加速查询。

但是有些情况下，用户的过滤条件无法匹配到前缀索引，比如 `where k3=3`。则无法通过索引提升查询速度。

创建以 k3 作为第一列的物化视图就可以解决这个问题。

**1）查询**

~~~sql
mysql> explain select record_id,seller_id,store_id from sales_records where store_id=3;
+-----------------------------------------------------------------------------------------+
| Explain String                                                                          |
+-----------------------------------------------------------------------------------------+
| PLAN FRAGMENT 0                                                                         |
|   OUTPUT EXPRS:                                                                         |
|     record_id[#0]                                                                       |
|     seller_id[#1]                                                                       |
|     store_id[#2]                                                                        |
|   PARTITION: UNPARTITIONED                                                              |
|                                                                                         |
|   VRESULT SINK                                                                          |
|                                                                                         |
|   1:VEXCHANGE                                                                           |
|      offset: 0                                                                          |
|                                                                                         |
| PLAN FRAGMENT 1                                                                         |
|                                                                                         |
|   PARTITION: HASH_PARTITIONED: record_id[#0]                                            |
|                                                                                         |
|   STREAM DATA SINK                                                                      |
|     EXCHANGE ID: 01                                                                     |
|     UNPARTITIONED                                                                       |
|                                                                                         |
|   0:VOlapScanNode                                                                       |
|      TABLE: default_cluster:example_db.sales_records(sales_records), PREAGGREGATION: ON |
|      PREDICATES: store_id[#2] = 3                                                       |
|      partitions=1/1, tablets=10/10, tabletList=12362,12364,12366 ...                    |
|      cardinality=1, avgRowSize=4625.0, numNodes=3                                       |
|      pushAggOp=NONE                                                                     |
+-----------------------------------------------------------------------------------------+
26 rows in set (0.03 sec)

~~~

**2）创建物化视图**

~~~sql
create materialized view mv_1 as 
select 
 store_id,
 record_id,
 seller_id,
 sale_date,
 sale_amt
from sales_records;
~~~

通过上面语法创建完成后，物化视图中既保留了完整的明细数据，且物化视图的前缀索 引为 store_id 列。

**3）查看表结构**

~~~sql
desc sales_records all;
~~~

**4）查询匹配**

~~~sql
mysql> explain select record_id,seller_id,store_id from sales_records where store_id=3;
+--------------------------------------------------------------------------------+
| Explain String                                                                 |
+--------------------------------------------------------------------------------+
| PLAN FRAGMENT 0                                                                |
|   OUTPUT EXPRS:                                                                |
|     mv_record_id[#1]                                                           |
|     mv_seller_id[#2]                                                           |
|     mv_store_id[#0]                                                            |
|   PARTITION: UNPARTITIONED                                                     |
|                                                                                |
|   VRESULT SINK                                                                 |
|                                                                                |
|   1:VEXCHANGE                                                                  |
|      offset: 0                                                                 |
|                                                                                |
| PLAN FRAGMENT 1                                                                |
|                                                                                |
|   PARTITION: HASH_PARTITIONED: mv_record_id[#1]                                |
|                                                                                |
|   STREAM DATA SINK                                                             |
|     EXCHANGE ID: 01                                                            |
|     UNPARTITIONED                                                              |
|                                                                                |
|   0:VOlapScanNode                                                              |
|      TABLE: default_cluster:example_db.sales_records(mv_1), PREAGGREGATION: ON |
|      PREDICATES: mv_store_id[#0] = 3                                           |
|      partitions=1/1, tablets=10/10, tabletList=12462,12464,12466 ...           |
|      cardinality=1, avgRowSize=4625.0, numNodes=3                              |
|      pushAggOp=NONE                                                            |
+--------------------------------------------------------------------------------+
26 rows in set (0.03 sec)
~~~

这时候查询就会直接从刚才创建的 mv_1 物化视图中读取数据。物化视图对 k3 是存在前缀索引的，查询效率也会提升。

##### 3.8.5.4、示例4

在`Doris 2.0`中，我们对物化视图所支持的表达式做了一些增强，本示例将主要体现新版本物化视图对各种表达式的支持和提前过滤。

1. 创建一个 Base 表并插入一些数据。

    ```sql
    create table d_table (
       k1 int null,
       k2 int not null,
       k3 bigint null,
       k4 date null
    )
    duplicate key (k1,k2,k3)
    distributed BY hash(k1) buckets 3
    properties("replication_num" = "1");
    
    insert into d_table select 1,1,1,'2020-02-20';
    insert into d_table select 2,2,2,'2021-02-20';
    insert into d_table select 3,-3,null,'2022-02-20';
    ```

2. 创建一些物化视图。

    ```sql
    create materialized view k1a2p2ap3ps as select abs(k1)+k2+1,sum(abs(k2+2)+k3+3) from d_table group by abs(k1)+k2+1;
    create materialized view kymd as select year(k4),month(k4) from d_table where year(k4) = 2020; 
    -- 提前用where表达式过滤以减少物化视图中的数据量。
    ```

3. 用一些查询测试是否成功命中物化视图。

    ```sql
    explain select abs(k1)+k2+1,sum(abs(k2+2)+k3+3) from d_table group by abs(k1)+k2+1; -- 命中k1a2p2ap3ps
    explain select bin(abs(k1)+k2+1),sum(abs(k2+2)+k3+3) from d_table group by bin(abs(k1)+k2+1); -- 命中k1a2p2ap3ps
    explain select year(k4),month(k4) from d_table; -- 无法命中物化视图，因为where条件不匹配
    explain select year(k4)+month(k4) from d_table where year(k4) = 2020; -- 命中kymd
    ```

#### 3.8.6、局限性

1. 如果删除语句的条件列，在物化视图中不存在，则不能进行删除操作。如果一定要删除数据，则需要先将物化视图删除，然后方可删除数据。
2. 单表上过多的物化视图会影响导入的效率：导入数据时，物化视图和 Base 表数据是同步更新的，如果一张表的物化视图表超过 10 张，则有可能导致导入速度很慢。这就像单次导入需要同时导入 10 张表数据是一样的。
3. 物化视图针对 Unique Key数据模型，只能改变列顺序，不能起到聚合的作用，所以在Unique Key模型上不能通过创建物化视图的方式对数据进行粗粒度聚合操作
4. 目前一些优化器对sql的改写行为可能会导致物化视图无法被命中，例如k1+1-1被改写成k1，between被改写成<=和>=，day被改写成dayofmonth，遇到这种情况需要手动调整下查询和物化视图的语句。

### 3.9、修改表

使用 ALTER TABLE 命令可以对表进行修改，包括 partition 、rollup、schema change、 rename 和 index 五种。语法：

~~~sql
ALTER TABLE [database.]table
alter_clause1[, alter_clause2, ...];
alter_clause 分为 partition 、rollup、schema change、rename 和 index 五种。
~~~

#### 3.9.1、 rename

~~~sql
-- 1）将名为 table1 的表修改为 table2
ALTER TABLE table1 RENAME table2;

-- 2）将表 example_table 中名为 rollup1 的 rollup index 修改为 rollup2
ALTER TABLE example_table RENAME ROLLUP rollup1 rollup2;

-- 3）将表 example_table 中名为 p1 的 partition 修改为 p2
ALTER TABLE example_table RENAME PARTITION p1 p2;
~~~

#### 3.9.2、partition

~~~sql
-- 1）增加分区, 使用默认分桶方式
-- 现有分区 [MIN, 2013-01-01)，增加分区 [2013-01-01, 2014-01-01)，
ALTER TABLE example_db.my_table ADD PARTITION p1 VALUES LESS THAN ("2014-01-01");

-- 2）增加分区，使用新的分桶数
ALTER TABLE example_db.my_table ADD PARTITION p1 VALUES LESS THAN ("2015-01-01") DISTRIBUTED BY HASH(k1) BUCKETS 20;

-- 3）增加分区，使用新的副本数
ALTER TABLE example_db.my_table ADD PARTITION p1 VALUES LESS THAN ("2015-01-01") ("replication_num"="1");

-- 4）修改分区副本数
ALTER TABLE example_db.my_table MODIFY PARTITION p1 SET("replication_num"="1");

-- 5）批量修改指定分区
ALTER TABLE example_db.my_table MODIFY PARTITION (p1, p2, p4) SET("in_memory"="true");

-- 6）批量修改所有分区
ALTER TABLE example_db.my_table MODIFY PARTITION (*) SET("storage_medium"="HDD");

-- 7）删除分区
ALTER TABLE example_db.my_table DROP PARTITION p1;

-- 8）增加一个指定上下界的分区
ALTER TABLE example_db.my_table ADD PARTITION p1 VALUES [("2014-01-01"), ("2014-02-01"));
~~~

#### 3.9.3、rollup

~~~sql
-- 1）创建 index: example_rollup_index，基于 base index（k1,k2,k3,v1,v2）。列式存储。
ALTER TABLE example_db.my_table ADD ROLLUP example_rollup_index(k1, k3, v1, v2);

-- 2）创建 index: example_rollup_index2，基于 example_rollup_index（k1,k3,v1,v2）
ALTER TABLE example_db.my_table ADD ROLLUP example_rollup_index2 (k1, v1) FROM example_rollup_index;

-- 3）创建 index: example_rollup_index3, 基于 base index (k1,k2,k3,v1), 自定义 rollup 超时时间一小时。
ALTER TABLE example_db.my_table ADD ROLLUP example_rollup_index(k1, k3, v1) PROPERTIES("timeout" = "3600");

-- 4）删除 index: example_rollup_index2
ALTER TABLE example_db.my_table DROP ROLLUP example_rollup_index2;
~~~

#### 3.9.4 、column

使用 ALTER TABLE 命令可以修改表的 Schema，包括如下修改：

语法：

```sql
ALTER TABLE [database.]table alter_clause;
```

schema change 的 alter_clause 支持如下几种修改方式：

1. 向指定 index 的指定位置添加一列

语法：

```sql
ADD COLUMN column_name column_type [KEY | agg_type] [DEFAULT "default_value"]
[AFTER column_name|FIRST]
[TO rollup_index_name]
[PROPERTIES ("key"="value", ...)]
```

注意：

- 聚合模型如果增加 value 列，需要指定 agg_type
- 非聚合模型（如 DUPLICATE KEY）如果增加key列，需要指定KEY关键字
- 不能在 rollup index 中增加 base index 中已经存在的列（如有需要，可以重新创建一个 rollup index）

1. 向指定 index 添加多列

语法：

```sql
ADD COLUMN (column_name1 column_type [KEY | agg_type] DEFAULT "default_value", ...)
[TO rollup_index_name]
[PROPERTIES ("key"="value", ...)]
```

注意：

- 聚合模型如果增加 value 列，需要指定agg_type
- 聚合模型如果增加key列，需要指定KEY关键字
- 不能在 rollup index 中增加 base index 中已经存在的列（如有需要，可以重新创建一个 rollup index）

1. 从指定 index 中删除一列

语法：

```sql
DROP COLUMN column_name
[FROM rollup_index_name]
```

注意：

- 不能删除分区列
- 如果是从 base index 中删除列，则如果 rollup index 中包含该列，也会被删除

1. 修改指定 index 的列类型以及列位置

    语法：

```sql
MODIFY COLUMN column_name column_type [KEY | agg_type] [NULL | NOT NULL] [DEFAULT "default_value"]
[AFTER column_name|FIRST]
[FROM rollup_index_name]
[PROPERTIES ("key"="value", ...)]
```

注意：

- 聚合模型如果修改 value 列，需要指定 agg_type
- 非聚合类型如果修改key列，需要指定KEY关键字
- 只能修改列的类型，列的其他属性维持原样（即其他属性需在语句中按照原属性显式的写出，参见 example 8）
- 分区列和分桶列不能做任何修改
- 目前支持以下类型的转换（精度损失由用户保证）
    - TINYINT/SMALLINT/INT/BIGINT/LARGEINT/FLOAT/DOUBLE 类型向范围更大的数字类型转换
    - TINTINT/SMALLINT/INT/BIGINT/LARGEINT/FLOAT/DOUBLE/DECIMAL 转换成 VARCHAR
    - VARCHAR 支持修改最大长度
    - VARCHAR/CHAR 转换成 TINTINT/SMALLINT/INT/BIGINT/LARGEINT/FLOAT/DOUBLE
    - VARCHAR/CHAR 转换成 DATE (目前支持"%Y-%m-%d", "%y-%m-%d", "%Y%m%d", "%y%m%d", "%Y/%m/%d, "%y/%m/%d"六种格式化格式)
    - DATETIME 转换成 DATE(仅保留年-月-日信息, 例如: `2019-12-09 21:47:05` <--> `2019-12-09`)
    - DATE 转换成 DATETIME(时分秒自动补零， 例如: `2019-12-09` <--> `2019-12-09 00:00:00`)
    - FLOAT 转换成 DOUBLE
    - INT 转换成 DATE (如果INT类型数据不合法则转换失败，原始数据不变)
    - 除DATE与DATETIME以外都可以转换成STRING，但是STRING不能转换任何其他类型

1. 对指定 index 的列进行重新排序

语法：

```sql
ORDER BY (column_name1, column_name2, ...)
[FROM rollup_index_name]
[PROPERTIES ("key"="value", ...)]
```

注意：

- index 中的所有列都要写出来
- value 列在 key 列之后

#### 3.9.5、更多命令

可以查看该网站，选择sql手册查看[快速开始 - Apache Doris](https://doris.apache.org/zh-CN/docs/sql-manual/sql-reference/Data-Definition-Statements/Alter/ALTER-TABLE-COLUMN)

## 四、数据导入和导出

### 4.1、数据导入

Doris 提供多种数据导入方案，可以针对不同的数据源进行选择不同的数据导入方式。

按场景划分

| 数据源                               | 导入方式                                                     |
| ------------------------------------ | ------------------------------------------------------------ |
| 对象存储（s3）,HDFS                  | [使用Broker导入数据](https://doris.apache.org/zh-CN/docs/data-operate/import/import-scenes/external-storage-load) |
| 本地文件                             | [导入本地数据](https://doris.apache.org/zh-CN/docs/data-operate/import/import-scenes/local-file-load) |
| Kafka                                | [订阅Kafka数据](https://doris.apache.org/zh-CN/docs/data-operate/import/import-scenes/kafka-load) |
| Mysql、PostgreSQL，Oracle，SQLServer | [通过外部表同步数据](https://doris.apache.org/zh-CN/docs/data-operate/import/import-scenes/external-table-load) |
| 通过JDBC导入                         | [使用JDBC同步数据](https://doris.apache.org/zh-CN/docs/data-operate/import/import-scenes/jdbc-load) |
| 导入JSON格式数据                     | [JSON格式数据导入](https://doris.apache.org/zh-CN/docs/data-operate/import/import-way/load-json-format) |

**按导入方式划分**

| 导入方式名称 | 使用方式                                                     |
| ------------ | ------------------------------------------------------------ |
| Spark Load   | [通过Spark导入外部数据](https://doris.apache.org/zh-CN/docs/data-operate/import/import-way/spark-load-manual) |
| Broker Load  | [通过Broker导入外部存储数据](https://doris.apache.org/zh-CN/docs/data-operate/import/import-way/broker-load-manual) |
| Stream Load  | [流式导入数据(本地文件及内存数据)](https://doris.apache.org/zh-CN/docs/data-operate/import/import-way/stream-load-manual) |
| Routine Load | [导入Kafka数据](https://doris.apache.org/zh-CN/docs/data-operate/import/import-way/routine-load-manual) |
| Insert Into  | [外部表通过INSERT方式导入数据](https://doris.apache.org/zh-CN/docs/data-operate/import/import-way/insert-into-manual) |
| S3 Load      | [S3协议的对象存储数据导入](https://doris.apache.org/zh-CN/docs/data-operate/import/import-way/s3-load-manual) |
| MySQL Load   | [MySQL客户端导入本地数据](https://doris.apache.org/zh-CN/docs/data-operate/import/import-way/mysql-load-manual) |

**支持的数据格式**

不同的导入方式支持的数据格式略有不同。

| 导入方式     | 支持的格式              |
| ------------ | ----------------------- |
| Broker Load  | parquet、orc、csv、gzip |
| Stream Load  | csv、json、parquet、orc |
| Routine Load | csv、json               |
| MySQL Load   | csv                     |

#### 4.1.1、Broker Load

Broker load 是一个异步的导入方式，支持的数据源取决于 [Broker](https://doris.apache.org/zh-CN/docs/advanced/broker) 进程支持的数据源。

因为 Doris 表里的数据是有序的，所以 Broker load 在导入数据的时是要利用doris 集群资源对数据进行排序，相对于 Spark load 来完成海量历史数据迁移，对 Doris 的集群资源占用要比较大，这种方式是在用户没有 Spark 这种计算资源的情况下使用，如果有 Spark 计算资源建议使用 [Spark load](https://doris.apache.org/zh-CN/docs/data-operate/import/import-way/spark-load-manual)。

##### 4.1.1.1、适用场景

- 源数据在 Broker 可以访问的存储系统中，如 HDFS。
- 数据量在 几十到百GB 级别。

##### 4.1.1.2、基本原理

用户在提交导入任务后，FE 会生成对应的 Plan 并根据目前 BE 的个数和文件的大小，将 Plan 分给 多个 BE 执行，每个 BE 执行一部分导入数据。

BE 在执行的过程中会从 Broker 拉取数据，在对数据 transform 之后将数据导入系统。所有 BE 均完成导入，由 FE 最终决定导入是否成功。

```
                 +
                 | 1. user create broker load
                 v
            +----+----+
            |         |
            |   FE    |
            |         |
            +----+----+
                 |
                 | 2. BE etl and load the data
    +--------------------------+
    |            |             |
+---v---+     +--v----+    +---v---+
|       |     |       |    |       |
|  BE   |     |  BE   |    |   BE  |
|       |     |       |    |       |
+---+-^-+     +---+-^-+    +--+-^--+
    | |           | |         | |
    | |           | |         | | 3. pull data from broker
+---v-+-+     +---v-+-+    +--v-+--+
|       |     |       |    |       |
|Broker |     |Broker |    |Broker |
|       |     |       |    |       |
+---+-^-+     +---+-^-+    +---+-^-+
    | |           | |          | |
+---v-+-----------v-+----------v-+-+
|       HDFS/BOS/AFS cluster       |
|                                  |
+----------------------------------+
```

##### 4.1.1.3、语法

~~~sql
LOAD LABEL load_label
(
data_desc1[, data_desc2, ...]
)
WITH BROKER broker_name
[broker_properties]
[load_properties]
[COMMENT "comments"];
~~~

- `load_label`

    每个导入需要指定一个唯一的 Label。后续可以通过这个 label 来查看作业进度。

    `[database.]label_name`

- `data_desc1`

    用于描述一组需要导入的文件。

    ```sql
    [MERGE|APPEND|DELETE]
    DATA INFILE
    (
    "file_path1"[, file_path2, ...]
    )
    [NEGATIVE]
    INTO TABLE `table_name`
    [PARTITION (p1, p2, ...)]
    [COLUMNS TERMINATED BY "column_separator"]
    [LINES TERMINATED BY "line_delimiter"]
    [FORMAT AS "file_type"]
    [COMPRESS_TYPE AS "compress_type"]
    [(column_list)]
    [COLUMNS FROM PATH AS (c1, c2, ...)]
    [SET (column_mapping)]
    [PRECEDING FILTER predicate]
    [WHERE predicate]
    [DELETE ON expr]
    [ORDER BY source_sequence]
    [PROPERTIES ("key1"="value1", ...)]
    ```

    - `[MERGE|APPEND|DELETE]`

        数据合并类型。默认为 APPEND，表示本次导入是普通的追加写操作。MERGE 和 DELETE 类型仅适用于 Unique Key 模型表。其中 MERGE 类型需要配合 `[DELETE ON]` 语句使用，以标注 Delete Flag 列。而 DELETE 类型则表示本次导入的所有数据皆为删除数据。

    - `DATA INFILE`

        指定需要导入的文件路径。可以是多个。可以使用通配符。路径最终必须匹配到文件，如果只匹配到目录则导入会失败。

    - `NEGATIVE`

        该关键词用于表示本次导入为一批”负“导入。这种方式仅针对具有整型 SUM 聚合类型的聚合数据表。该方式会将导入数据中，SUM 聚合列对应的整型数值取反。主要用于冲抵之前导入错误的数据。

    - `PARTITION(p1, p2, ...)`

        可以指定仅导入表的某些分区。不在分区范围内的数据将被忽略。

    - `COLUMNS TERMINATED BY`

        指定列分隔符。仅在 CSV 格式下有效。仅能指定单字节分隔符。

    - `LINES TERMINATED BY`

        指定行分隔符。仅在 CSV 格式下有效。仅能指定单字节分隔符。

    - `FORMAT AS`

        指定文件类型，支持 CSV、PARQUET 和 ORC 格式。默认为 CSV。

    - `COMPRESS_TYPE AS` 指定文件压缩类型, 支持GZ/BZ2/LZ4FRAME。

    - `column list`

        用于指定原始文件中的列顺序。关于这部分详细介绍，可以参阅 [列的映射，转换与过滤](https://doris.apache.org/zh-CN/docs/data-operate/import/import-scenes/load-data-convert) 文档。

        `(k1, k2, tmpk1)`

    - `COLUMNS FROM PATH AS`

        指定从导入文件路径中抽取的列。

    - `SET (column_mapping)`

        指定列的转换函数。

    - `PRECEDING FILTER predicate`

        前置过滤条件。数据首先根据 `column list` 和 `COLUMNS FROM PATH AS` 按顺序拼接成原始数据行。然后按照前置过滤条件进行过滤。关于这部分详细介绍，可以参阅 [列的映射，转换与过滤](https://doris.apache.org/zh-CN/docs/data-operate/import/import-scenes/load-data-convert) 文档。

    - `WHERE predicate`

        根据条件对导入的数据进行过滤。关于这部分详细介绍，可以参阅 [列的映射，转换与过滤](https://doris.apache.org/zh-CN/docs/data-operate/import/import-scenes/load-data-convert) 文档。

    - `DELETE ON expr`

        需配合 MEREGE 导入模式一起使用，仅针对 Unique Key 模型的表。用于指定导入数据中表示 Delete Flag 的列和计算关系。

    - `ORDER BY`

        仅针对 Unique Key 模型的表。用于指定导入数据中表示 Sequence Col 的列。主要用于导入时保证数据顺序。

    - `PROPERTIES ("key1"="value1", ...)`

        指定导入的format的一些参数。如导入的文件是`json`格式，则可以在这里指定`json_root`、`jsonpaths`、`fuzzy_parse`等参数。

        - enclose

            包围符。当csv数据字段中含有行分隔符或列分隔符时，为防止意外截断，可指定单字节字符作为包围符起到保护作用。例如列分隔符为","，包围符为"'"，数据为"a,'b,c'",则"b,c"会被解析为一个字段。

        - escape

            转义符。用于转义在字段中出现的与包围符相同的字符。例如数据为"a,'b,'c'"，包围符为"'"，希望"b,'c被作为一个字段解析，则需要指定单字节转义符，例如"\"，然后将数据修改为"a,'b,\'c'"。

- `WITH BROKER broker_name`

    指定需要使用的 Broker 服务名称。在公有云 Doris 中。Broker 服务名称为 `bos`

- `broker_properties`

    指定 broker 所需的信息。这些信息通常被用于 Broker 能够访问远端存储系统。如 BOS 或 HDFS。关于具体信息，可参阅 [Broker](https://doris.apache.org/zh-CN/docs/advanced/broker) 文档。

    ```text
    (
        "key1" = "val1",
        "key2" = "val2",
        ...
    )
    ```

    - `load_properties`

        指定导入的相关参数。目前支持以下参数：

        - `timeout`

            导入超时时间。默认为 4 小时。单位秒。

        - `max_filter_ratio`

            最大容忍可过滤（数据不规范等原因）的数据比例。默认零容忍。取值范围为 0 到 1。

        - `exec_mem_limit`

            导入内存限制。默认为 2GB。单位为字节。

        - `strict_mode`

            是否对数据进行严格限制。默认为 false。

        - `partial_columns`

            布尔类型，为 true 表示使用部分列更新，默认值为 false，该参数只允许在表模型为 Unique 且采用 Merge on Write 时设置。

        - `timezone`

            指定某些受时区影响的函数的时区，如 `strftime/alignment_timestamp/from_unixtime` 等等，具体请查阅 [时区](https://doris.apache.org/zh-CN/docs/advanced/time-zone) 文档。如果不指定，则使用 "Asia/Shanghai" 时区

        - `load_parallelism`

            导入并发度，默认为1。调大导入并发度会启动多个执行计划同时执行导入任务，加快导入速度。

        - `send_batch_parallelism`

            用于设置发送批处理数据的并行度，如果并行度的值超过 BE 配置中的 `max_send_batch_parallelism_per_job`，那么作为协调点的 BE 将使用 `max_send_batch_parallelism_per_job` 的值。

        - `load_to_single_tablet`

            布尔类型，为true表示支持一个任务只导入数据到对应分区的一个tablet，默认值为false，作业的任务数取决于整体并发度。该参数只允许在对带有random分桶的olap表导数的时候设置。

        - priority

            设置导入任务的优先级，可选 `HIGH/NORMAL/LOW` 三种优先级，默认为 `NORMAL`，对于处在 `PENDING` 状态的导入任务，更高优先级的任务将优先被执行进入 `LOADING` 状态。

- comment

    指定导入任务的备注信息。可选参数。

##### 4.1.1.4、CSV导入示例

**1）建表**

~~~sql
create table example_db.student_broker_load_01
(
    id int,
    name varchar(50),
    age int,
    course_id int,
    score decimal(10,4)
)
DUPLICATE KEY(id)
DISTRIBUTED BY HASH(id) BUCKETS 10;
~~~

**2）启动hdfs**

~~~shell
hdfs dfs -put /doris/load/broker_load/01/studne.csv
~~~

**3）创建导入任务**

~~~sql
load label example_db.student_broker_load_01 (
    data infile("hdfs://hadoop102:8020/doris/load/broker_load/broker_load_01.csv")
	into table `student_broker_load_01`
    columns terminated by ','
    lines terminated by '\n'
    format as 'csv'
    (id,name,age,course_id,score)
)
with broker hdfs_broker (
	"username" = "hadoop",
    "password" = "123456"
);
~~~

**4）更多的导入示例**

csv文件的导入与传统的text文件导入配置并没有什么区别，format as 'csv'配置不写也可以正常导入数据，想要了解更多的Broker load示例请查看 [BROKER_LOAD](https://doris.apache.org/zh-CN/docs/sql-manual/sql-reference/Data-Manipulation-Statements/Load/BROKER-LOAD/#example)

##### 4.1.1.5、查看load任务

由于Broker Load 是一个异步导入过程，语句执行成功仅代表导入任务提交成功，并不代表数据导入成功。导入状态需要通过 [SHOW LOAD](https://doris.apache.org/zh-CN/docs/sql-manual/sql-reference/Show-Statements/SHOW-LOAD?_highlight=show&_highlight=load) 命令查看。

~~~sql
mysql> show load from example_db\G
*************************** 1. row ***************************
         JobId: 13268
         Label: student_broker_load_01
         State: FINISHED
      Progress: 100.00% (1/1)
          Type: BROKER
       EtlInfo: unselected.rows=0; dpp.abnorm.ALL=0; dpp.norm.ALL=4
      TaskInfo: cluster:hdfs_broker; timeout(s):14400; max_filter_ratio:0.0; priority:NORMAL
      ErrorMsg: NULL
    CreateTime: 2023-11-24 14:28:46
  EtlStartTime: 2023-11-24 14:28:48
 EtlFinishTime: 2023-11-24 14:28:48
 LoadStartTime: 2023-11-24 14:28:48
LoadFinishTime: 2023-11-24 14:28:49
           URL: NULL
    JobDetails: {"Unfinished backends":{"2356ce5855a04628-82276d5815027cbf":[]},"ScannedRows":4,"TaskNumber":1,"LoadBytes":140,"All backends":{"2356ce5855a04628-82276d5815027cbf":[10094]},"FileNumber":1,"FileSize":68}
 TransactionId: 3056
  ErrorTablets: {}
          User: root
       Comment: 
1 row in set (0.00 sec)
~~~

##### 4.1.1.6、取消load任务

已提交切尚未结束的导入任务可以通过 [CANCEL LOAD](https://doris.apache.org/zh-CN/docs/sql-manual/sql-reference/Data-Manipulation-Statements/Load/CANCEL-LOAD?_highlight=cancel&_highlight=load) 命令取消。取消后，已写入的数据也会回滚，不会生效。

~~~sql
cancel load from example_db where label = "label_name";
~~~

#### 4.1.2、Stream Load

Stream load 是一个同步的导入方式，用户通过发送 HTTP 协议发送请求将本地文件或数据流导入到 Doris 中。Stream load 同步执行导入并返回导入结果。用户可直接通过请求的返回体判断本次导入是否成功。

Stream load 主要适用于导入本地文件，或通过程序导入数据流中的数据。

##### 4.1.2.1、基本原理

下图展示了 Stream load 的主要流程，省略了一些导入细节。

```text
                         ^      +
                         |      |
                         |      | 1A. User submit load to FE
                         |      |
                         |   +--v-----------+
                         |   | FE           |
4. Return result to user |   +--+-----------+
                         |      |
                         |      | 2. Redirect to BE
                         |      |
                         |   +--v-----------+
                         +---+Coordinator BE| 1B. User submit load to BE
                             +-+-----+----+-+
                               |     |    |
                         +-----+     |    +-----+
                         |           |          | 3. Distrbute data
                         |           |          |
                       +-v-+       +-v-+      +-v-+
                       |BE |       |BE |      |BE |
                       +---+       +---+      +---+
```

Stream load 中，Doris 会选定一个节点作为 Coordinator 节点。该节点负责接数据并分发数据到其他数据节点。

用户通过 HTTP 协议提交导入命令。如果提交到 FE，则 FE 会通过 HTTP redirect 指令将请求转发给某一个 BE。用户也可以直接提交导入命令给某一指定 BE。

导入的最终结果由 Coordinator BE 返回给用户。

##### 4.1.2.2、支持数据格式

目前 Stream Load 支持数据格式：CSV（文本）、JSON

Doris 1.2.0之后新增支持：PARQUET 、ORC

##### 4.1.2.3、基本操作

Stream Load 通过 HTTP 协议提交和传输数据。这里通过 `curl` 命令展示如何提交导入。

用户也可以通过其他 HTTP client 进行操作。

```shell
curl --location-trusted -u user:passwd [-H ""...] -T data.file -XPUT http://fe_host:http_port/api/{db}/{table}/_stream_load

# Header 中支持属性见下面的 ‘导入任务参数’ 说明
# 格式为: -H "key1:value1"
```

示例：

```shell
curl --location-trusted -u root -T date -H "label:123" http://abc.com:8030/api/test/date/_stream_load
```

创建导入的详细语法帮助执行 `HELP STREAM LOAD` 查看, 下面主要介绍创建 Stream Load 的部分参数意义。

**签名参数**

- user/passwd

    Stream load 由于创建导入的协议使用的是 HTTP 协议，通过 Basic access authentication 进行签名。Doris 系统会根据签名验证用户身份和导入权限。

**导入任务参数**

Stream Load 由于使用的是 HTTP 协议，所以所有导入任务有关的参数均设置在 Header 中。下面主要介绍了 Stream Load 导入任务参数的部分参数意义。

- label

    导入任务的标识。每个导入任务，都有一个在单 database 内部唯一的 label。label 是用户在导入命令中自定义的名称。通过这个 label，用户可以查看对应导入任务的执行情况。

    label 的另一个作用，是防止用户重复导入相同的数据。**强烈推荐用户同一批次数据使用相同的 label。这样同一批次数据的重复请求只会被接受一次，保证了 At-Most-Once**

    当 label 对应的导入作业状态为 CANCELLED 时，该 label 可以再次被使用。

- column_separator

    用于指定导入文件中的列分隔符，默认为\t。如果是不可见字符，则需要加\x作为前缀，使用十六进制来表示分隔符。

    如hive文件的分隔符\x01，需要指定为-H "column_separator:\x01"。

    可以使用多个字符的组合作为列分隔符。

- line_delimiter

    用于指定导入文件中的换行符，默认为\n。

    可以使用做多个字符的组合作为换行符。

- max_filter_ratio

    导入任务的最大容忍率，默认为0容忍，取值范围是0~1。当导入的错误率超过该值，则导入失败。

    如果用户希望忽略错误的行，可以通过设置这个参数大于 0，来保证导入可以成功。

    计算公式为：

    `(dpp.abnorm.ALL / (dpp.abnorm.ALL + dpp.norm.ALL ) ) > max_filter_ratio`

    `dpp.abnorm.ALL` 表示数据质量不合格的行数。如类型不匹配，列数不匹配，长度不匹配等等。

    `dpp.norm.ALL` 指的是导入过程中正确数据的条数。可以通过 `SHOW LOAD` 命令查询导入任务的正确数据量。

    原始文件的行数 = `dpp.abnorm.ALL + dpp.norm.ALL`

- where

    导入任务指定的过滤条件。Stream load 支持对原始数据指定 where 语句进行过滤。被过滤的数据将不会被导入，也不会参与 filter ratio 的计算，但会被计入`num_rows_unselected`。

- Partitions

    待导入表的 Partition 信息，如果待导入数据不属于指定的 Partition 则不会被导入。这些数据将计入 `dpp.abnorm.ALL`

- columns

    待导入数据的函数变换配置，目前 Stream load 支持的函数变换方法包含列的顺序变化以及表达式变换，其中表达式变换的方法与查询语句的一致。

- format

    指定导入数据格式，支持 `csv` 和 `json`，默认是 `csv`

    Doris 1.2 支持 `csv_with_names` (csv文件行首过滤)、`csv_with_names_and_types`(csv文件前两行过滤)、`parquet`、`orc`

    ```text
    列顺序变换例子：原始数据有三列(src_c1,src_c2,src_c3), 目前doris表也有三列（dst_c1,dst_c2,dst_c3）
    
    如果原始表的src_c1列对应目标表dst_c1列，原始表的src_c2列对应目标表dst_c2列，原始表的src_c3列对应目标表dst_c3列，则写法如下：
    columns: dst_c1, dst_c2, dst_c3
    
    如果原始表的src_c1列对应目标表dst_c2列，原始表的src_c2列对应目标表dst_c3列，原始表的src_c3列对应目标表dst_c1列，则写法如下：
    columns: dst_c2, dst_c3, dst_c1
    
    表达式变换例子：原始文件有两列，目标表也有两列（c1,c2）但是原始文件的两列均需要经过函数变换才能对应目标表的两列，则写法如下：
    columns: tmp_c1, tmp_c2, c1 = year(tmp_c1), c2 = month(tmp_c2)
    其中 tmp_*是一个占位符，代表的是原始文件中的两个原始列。
    ```

- exec_mem_limit

    导入内存限制。默认为 2GB，单位为字节。

- strict_mode

    Stream Load 导入可以开启 strict mode 模式。开启方式为在 HEADER 中声明 `strict_mode=true` 。默认的 strict mode 为关闭。

    strict mode 模式的意思是：对于导入过程中的列类型转换进行严格过滤。严格过滤的策略如下：

    1. 对于列类型转换来说，如果 strict mode 为true，则错误的数据将被 filter。这里的错误数据是指：原始数据并不为空值，在参与列类型转换后结果为空值的这一类数据。
    2. 对于导入的某列由函数变换生成时，strict mode 对其不产生影响。
    3. 对于导入的某列类型包含范围限制的，如果原始数据能正常通过类型转换，但无法通过范围限制的，strict mode 对其也不产生影响。例如：如果类型是 decimal(1,0), 原始数据为 10，则属于可以通过类型转换但不在列声明的范围内。这种数据 strict 对其不产生影响。

- merge_type

    数据的合并类型，一共支持三种类型APPEND、DELETE、MERGE 其中，APPEND是默认值，表示这批数据全部需要追加到现有数据中，DELETE 表示删除与这批数据key相同的所有行，MERGE 语义 需要与delete 条件联合使用，表示满足delete 条件的数据按照DELETE 语义处理其余的按照APPEND 语义处理

- two_phase_commit

    Stream load 导入可以开启两阶段事务提交模式：在Stream load过程中，数据写入完成即会返回信息给用户，此时数据不可见，事务状态为`PRECOMMITTED`，用户手动触发commit操作之后，数据才可见。

- enclose

    包围符。当csv数据字段中含有行分隔符或列分隔符时，为防止意外截断，可指定单字节字符作为包围符起到保护作用。例如列分隔符为","，包围符为"'"，数据为"a,'b,c'",则"b,c"会被解析为一个字段。

- escape

    转义符。用于转义在csv字段中出现的与包围符相同的字符。例如数据为"a,'b,'c'"，包围符为"'"，希望"b,'c被作为一个字段解析，则需要指定单字节转义符，例如"\"，然后将数据修改为"a,'b,\'c'"。

    示例：

    1. 发起stream load预提交操作

    ```shell
    curl  --location-trusted -u user:passwd -H "two_phase_commit:true" -T test.txt http://fe_host:http_port/api/{db}/{table}/_stream_load
    {
        "TxnId": 18036,
        "Label": "55c8ffc9-1c40-4d51-b75e-f2265b3602ef",
        "TwoPhaseCommit": "true",
        "Status": "Success",
        "Message": "OK",
        "NumberTotalRows": 100,
        "NumberLoadedRows": 100,
        "NumberFilteredRows": 0,
        "NumberUnselectedRows": 0,
        "LoadBytes": 1031,
        "LoadTimeMs": 77,
        "BeginTxnTimeMs": 1,
        "StreamLoadPutTimeMs": 1,
        "ReadDataTimeMs": 0,
        "WriteDataTimeMs": 58,
        "CommitAndPublishTimeMs": 0
    }
    ```

    1. 对事务触发commit操作 注意1) 请求发往fe或be均可 注意2) commit 的时候可以省略 url 中的 `{table}` 使用事务id

    ```shell
    curl -X PUT --location-trusted -u user:passwd  -H "txn_id:18036" -H "txn_operation:commit"  http://fe_host:http_port/api/{db}/{table}/_stream_load_2pc
    {
        "status": "Success",
        "msg": "transaction [18036] commit successfully."
    }
    ```

    使用label

    ```shell
    curl -X PUT --location-trusted -u user:passwd  -H "label:55c8ffc9-1c40-4d51-b75e-f2265b3602ef" -H "txn_operation:commit"  http://fe_host:http_port/api/{db}/{table}/_stream_load_2pc
    {
        "status": "Success",
        "msg": "label [55c8ffc9-1c40-4d51-b75e-f2265b3602ef] commit successfully."
    }
    ```

    1. 对事务触发abort操作 注意1) 请求发往fe或be均可 注意2) abort 的时候可以省略 url 中的 `{table}` 使用事务id

    ```shell
    curl -X PUT --location-trusted -u user:passwd  -H "txn_id:18037" -H "txn_operation:abort"  http://fe_host:http_port/api/{db}/{table}/_stream_load_2pc
    {
        "status": "Success",
        "msg": "transaction [18037] abort successfully."
    }
    ```

    使用label

    ```shell
    curl -X PUT --location-trusted -u user:passwd  -H "label:55c8ffc9-1c40-4d51-b75e-f2265b3602ef" -H "txn_operation:abort"  http://fe_host:http_port/api/{db}/{table}/_stream_load_2pc
    {
        "status": "Success",
        "msg": "label [55c8ffc9-1c40-4d51-b75e-f2265b3602ef] abort successfully."
    }
    ```

- enable_profile

    SinceVersion 1.2.7当 `enable_profile` 为 true 时，Stream Load profile 将会被打印到 be.INFO 日志中。

- memtable_on_sink_node

    SinceVersion 2.1.0是否在数据导入中启用 MemTable 前移，默认为 false

    在 DataSink 节点上构建 MemTable，并通过 brpc streaming 发送 segment 到其他 BE。 该方法减少了多副本之间的重复工作，并且节省了数据序列化和反序列化的时间。

- partial_columns

    SinceVersion 2.0

    是否启用部分列更新，布尔类型，为 true 表示使用部分列更新，默认值为 false，该参数只允许在表模型为 Unique 且采用 Merge on Write 时设置。

    eg: `curl --location-trusted -u root: -H "partial_columns:true" -H "column_separator:," -H "columns:id,balance,last_access_time" -T /tmp/test.csv http://127.0.0.1:48037/api/db1/user_profile/_stream_load`

##### 4.1.2.4、导入示例

**创建导入任务**

~~~shell
curl --location-trusted -u root -H "label:123" -H "timeout:100" -H "column_separator:," -T broker_load_02.csv http://hadoop102:8030/api/example_db/student_broker_load_01/_stream_load
~~~

##### 4.1.2.5、返回结果

由于 Stream load 是一种同步的导入方式，所以导入的结果会通过创建导入的返回值直接返回给用户。

~~~shell
[hadoop@hadoop102 data]$ curl --location-trusted -u root -H "label:123" -H "timeout:100" -H "column_separator:," -T broker_load_02.csv http://hadoop102:8030/api/example_db/student_broker_load_01/_stream_load
Enter host password for user 'root':
{
    "TxnId": 3057,
    "Label": "123",
    "Comment": "",
    "TwoPhaseCommit": "false",
    "Status": "Success",
    "Message": "OK",
    "NumberTotalRows": 2,
    "NumberLoadedRows": 2,
    "NumberFilteredRows": 0,
    "NumberUnselectedRows": 0,
    "LoadBytes": 34,
    "LoadTimeMs": 138,
    "BeginTxnTimeMs": 24,
    "StreamLoadPutTimeMs": 30,
    "ReadDataTimeMs": 0,
    "WriteDataTimeMs": 25,
    "CommitAndPublishTimeMs": 56
}
[hadoop@hadoop102 data]$
~~~

下面主要解释了 Stream load 导入结果参数：

- TxnId：导入的事务ID。用户可不感知。

- Label：导入 Label。由用户指定或系统自动生成。

- Status：导入完成状态。

    "Success"：表示导入成功。

    "Publish Timeout"：该状态也表示导入已经完成，只是数据可能会延迟可见，无需重试。

    "Label Already Exists"：Label 重复，需更换 Label。

    "Fail"：导入失败。

- ExistingJobStatus：已存在的 Label 对应的导入作业的状态。

    这个字段只有在当 Status 为 "Label Already Exists" 时才会显示。用户可以通过这个状态，知晓已存在 Label 对应的导入作业的状态。"RUNNING" 表示作业还在执行，"FINISHED" 表示作业成功。

- Message：导入错误信息。

- NumberTotalRows：导入总处理的行数。

- NumberLoadedRows：成功导入的行数。

- NumberFilteredRows：数据质量不合格的行数。

- NumberUnselectedRows：被 where 条件过滤的行数。

- LoadBytes：导入的字节数。

- LoadTimeMs：导入完成时间。单位毫秒。

- BeginTxnTimeMs：向Fe请求开始一个事务所花费的时间，单位毫秒。

- StreamLoadPutTimeMs：向Fe请求获取导入数据执行计划所花费的时间，单位毫秒。

- ReadDataTimeMs：读取数据所花费的时间，单位毫秒。

- WriteDataTimeMs：执行写入数据操作所花费的时间，单位毫秒。

- CommitAndPublishTimeMs：向Fe请求提交并且发布事务所花费的时间，单位毫秒。

- ErrorURL：如果有数据质量问题，通过访问这个 URL 查看具体错误行。

> 注意：由于 Stream load 是同步的导入方式，所以并不会在 Doris 系统中记录导入信息，用户无法异步的通过查看导入命令看到 Stream load。使用时需监听创建导入请求的返回值获取导入结果。

##### 4.1.2.6、取消导入

用户无法手动取消 Stream Load，Stream Load 在超时或者导入错误后会被系统自动取消。

##### 4.1.2.7、查看 Stream Load

用户可以通过 `show stream load` 来查看已经完成的 stream load 任务，具体的使用可以查看[Stream Load](https://doris.apache.org/zh-CN/docs/sql-manual/sql-reference/Show-Statements/SHOW-STREAM-LOAD?_highlight=show&_highlight=st#description)。

默认 BE 是不记录 Stream Load 的记录，如果你要查看需要在 BE 上启用记录，配置参数是：`enable_stream_load_record=true` ，具体怎么配置请参照 [BE 配置项](https://doris.apache.org/zh-CN/docs/admin-manual/config/be-config)。

#### 4.1.3、Routine Load

例行导入（Routine Load）功能，支持用户提交一个常驻的导入任务，通过不断的从指定的数据源读取数据，将数据导入到 Doris 中。

##### 4.1.3.1、基本原理

```text
         +---------+
         |  Client |
         +----+----+
              |
+-----------------------------+
| FE          |               |
| +-----------v------------+  |
| |                        |  |
| |   Routine Load Job     |  |
| |                        |  |
| +---+--------+--------+--+  |
|     |        |        |     |
| +---v--+ +---v--+ +---v--+  |
| | task | | task | | task |  |
| +--+---+ +---+--+ +---+--+  |
|    |         |        |     |
+-----------------------------+
     |         |        |
     v         v        v
 +---+--+   +--+---+   ++-----+
 |  BE  |   |  BE  |   |  BE  |
 +------+   +------+   +------+
```

如上图，Client 向 FE 提交一个Routine Load 作业。

1. FE 通过 JobScheduler 将一个导入作业拆分成若干个 Task。每个 Task 负责导入指定的一部分数据。Task 被 TaskScheduler 分配到指定的 BE 上执行。
2. 在 BE 上，一个 Task 被视为一个普通的导入任务，通过 Stream Load 的导入机制进行导入。导入完成后，向 FE 汇报。
3. FE 中的 JobScheduler 根据汇报结果，继续生成后续新的 Task，或者对失败的 Task 进行重试。
4. 整个 Routine Load 作业通过不断的产生新的 Task，来完成数据不间断的导入。

##### 4.1.3.2、Kafka例行导入

当前我们仅支持从 Kafka 进行例行导入。该部分会详细介绍 Kafka 例行导入使用方式和最佳实践。

**使用限制**

1. 支持无认证的 Kafka 访问，以及通过 SSL 方式认证的 Kafka 集群。
2. 支持的消息格式为 csv， json 文本格式。csv 每一个 message 为一行，且行尾**不包含**换行符。
3. 默认支持 Kafka 0.10.0.0(含) 以上版本。如果要使用 Kafka 0.10.0.0 以下版本 (0.9.0, 0.8.2, 0.8.1, 0.8.0)，需要修改 be 的配置，将 kafka_broker_version_fallback 的值设置为要兼容的旧版本，或者在创建routine load的时候直接设置 property.broker.version.fallback的值为要兼容的旧版本，使用旧版本的代价是routine load 的部分新特性可能无法使用，如根据时间设置 kafka 分区的 offset。

##### 4.1.3.3、创建任务

创建例行导入任务的详细语法可以连接到 Doris 后，查看[CREATE ROUTINE LOAD](https://doris.apache.org/zh-CN/docs/sql-manual/sql-reference/Data-Manipulation-Statements/Load/CREATE-ROUTINE-LOAD)命令手册，或者执行 `HELP ROUTINE LOAD;` 查看语法帮助。

语法：

```sql
CREATE ROUTINE LOAD [db.]job_name [ON tbl_name]
[merge_type]
[load_properties]
[job_properties]
FROM data_source [data_source_properties]
[COMMENT "comment"]
```

~~~text
- `[db.]job_name`

  导入作业的名称，在同一个 database 内，相同名称只能有一个 job 在运行。

- `tbl_name` 

  指定需要导入的表的名称，可选参数，如果不指定，则采用动态表的方式，这个时候需要 Kafka 中的数据包含表名的信息。
  目前仅支持从 Kafka 的 Value 中获取表名，且需要符合这种格式：以 json 为例：`table_name|{"col1": "val1", "col2": "val2"}`, 
  其中 `tbl_name` 为表名，以 `|` 作为表名和表数据的分隔符。csv 格式的数据也是类似的，如：`table_name|val1,val2,val3`。注意，这里的 
  `table_name` 必须和 Doris 中的表名一致，否则会导致导入失败.
  
   tips: 动态表不支持 `columns_mapping` 参数。如果你的表结构和 Doris 中的表结构一致，且存在大量的表信息需要导入，那么这种方式将是不二选择。

- `merge_type`

  数据合并类型。默认为 APPEND，表示导入的数据都是普通的追加写操作。MERGE 和 DELETE 类型仅适用于 Unique Key 模型表。其中 MERGE 类型需要配合 [DELETE ON] 语句使用，以标注 Delete Flag 列。而 DELETE 类型则表示导入的所有数据皆为删除数据。
  tips: 当使用动态多表的时候，请注意此参数应该符合每张动态表的类型，否则会导致导入失败。

- load_properties

  用于描述导入数据。组成如下：

  ```SQL
  [column_separator],
  [columns_mapping],
  [preceding_filter],
  [where_predicates],
  [partitions],
  [DELETE ON],
  [ORDER BY]
~~~

- `column_separator`

    指定列分隔符，默认为 `\t`

    `COLUMNS TERMINATED BY ","`

- `columns_mapping`

    用于指定文件列和表中列的映射关系，以及各种列转换等。关于这部分详细介绍，可以参阅 [列的映射，转换与过滤] 文档。

    `(k1, k2, tmpk1, k3 = tmpk1 + 1)`

    tips: 动态表不支持此参数。

- `preceding_filter`

    过滤原始数据。关于这部分详细介绍，可以参阅 [列的映射，转换与过滤] 文档。

    tips: 动态表不支持此参数。

- `where_predicates`

    根据条件对导入的数据进行过滤。关于这部分详细介绍，可以参阅 [列的映射，转换与过滤] 文档。

    `WHERE k1 > 100 and k2 = 1000`

    tips: 当使用动态多表的时候，请注意此参数应该符合每张动态表的列，否则会导致导入失败。通常在使用动态多表的时候，我们仅建议通用公共列使用此参数。

- `partitions`

    指定导入目的表的哪些 partition 中。如果不指定，则会自动导入到对应的 partition 中。

    `PARTITION(p1, p2, p3)`

    tips: 当使用动态多表的时候，请注意此参数应该符合每张动态表，否则会导致导入失败。

- `DELETE ON`

    需配合 MEREGE 导入模式一起使用，仅针对 Unique Key 模型的表。用于指定导入数据中表示 Delete Flag 的列和计算关系。

    `DELETE ON v3 >100`

    tips: 当使用动态多表的时候，请注意此参数应该符合每张动态表，否则会导致导入失败。

- `ORDER BY`

    仅针对 Unique Key 模型的表。用于指定导入数据中表示 Sequence Col 的列。主要用于导入时保证数据顺序。

    tips: 当使用动态多表的时候，请注意此参数应该符合每张动态表，否则会导致导入失败。

- `job_properties`

    用于指定例行导入作业的通用参数。

    ```text
    PROPERTIES (
        "key1" = "val1",
        "key2" = "val2"
    )
    ```

    目前我们支持以下参数：

    1. `desired_concurrent_number`

        期望的并发度。一个例行导入作业会被分成多个子任务执行。这个参数指定一个作业最多有多少任务可以同时执行。必须大于0。默认为5。

        这个并发度并不是实际的并发度，实际的并发度，会通过集群的节点数、负载情况，以及数据源的情况综合考虑。

        `"desired_concurrent_number" = "3"`

    2. `max_batch_interval/max_batch_rows/max_batch_size`

        这三个参数分别表示：

        1. 每个子任务最大执行时间，单位是秒。范围为 1 到 60。默认为10。
        2. 每个子任务最多读取的行数。必须大于等于200000。默认是200000。
        3. 每个子任务最多读取的字节数。单位是字节，范围是 100MB 到 1GB。默认是 100MB。

        这三个参数，用于控制一个子任务的执行时间和处理量。当任意一个达到阈值，则任务结束。

        ```text
        "max_batch_interval" = "20",
        "max_batch_rows" = "300000",
        "max_batch_size" = "209715200"
        ```

    3. `max_error_number`

        采样窗口内，允许的最大错误行数。必须大于等于0。默认是 0，即不允许有错误行。

        采样窗口为 `max_batch_rows * 10`。即如果在采样窗口内，错误行数大于 `max_error_number`，则会导致例行作业被暂停，需要人工介入检查数据质量问题。

        被 where 条件过滤掉的行不算错误行。

    4. `strict_mode`

        是否开启严格模式，默认为关闭。如果开启后，非空原始数据的列类型变换如果结果为 NULL，则会被过滤。指定方式为：

        `"strict_mode" = "true"`

        strict mode 模式的意思是：对于导入过程中的列类型转换进行严格过滤。严格过滤的策略如下：

        1. 对于列类型转换来说，如果 strict mode 为true，则错误的数据将被 filter。这里的错误数据是指：原始数据并不为空值，在参与列类型转换后结果为空值的这一类数据。
        2. 对于导入的某列由函数变换生成时，strict mode 对其不产生影响。
        3. 对于导入的某列类型包含范围限制的，如果原始数据能正常通过类型转换，但无法通过范围限制的，strict mode 对其也不产生影响。例如：如果类型是 decimal(1,0), 原始数据为 10，则属于可以通过类型转换但不在列声明的范围内。这种数据 strict 对其不产生影响。

        **strict mode 与 source data 的导入关系**

        这里以列类型为 TinyInt 来举例

        > 注：当表中的列允许导入空值时

        | source data | source data example | string to int | strict_mode   | result                 |
        | ----------- | ------------------- | ------------- | ------------- | ---------------------- |
        | 空值        | \N                  | N/A           | true or false | NULL                   |
        | not null    | aaa or 2000         | NULL          | true          | invalid data(filtered) |
        | not null    | aaa                 | NULL          | false         | NULL                   |
        | not null    | 1                   | 1             | true or false | correct data           |

        这里以列类型为 Decimal(1,0) 举例

        > 注：当表中的列允许导入空值时

        | source data | source data example | string to int | strict_mode   | result                 |
        | ----------- | ------------------- | ------------- | ------------- | ---------------------- |
        | 空值        | \N                  | N/A           | true or false | NULL                   |
        | not null    | aaa                 | NULL          | true          | invalid data(filtered) |
        | not null    | aaa                 | NULL          | false         | NULL                   |
        | not null    | 1 or 10             | 1             | true or false | correct data           |

        > 注意：10 虽然是一个超过范围的值，但是因为其类型符合 decimal的要求，所以 strict mode对其不产生影响。10 最后会在其他 ETL 处理流程中被过滤。但不会被 strict mode 过滤。

    5. `timezone`

        指定导入作业所使用的时区。默认为使用 Session 的 timezone 参数。该参数会影响所有导入涉及的和时区有关的函数结果。

    6. `format`

        指定导入数据格式，默认是csv，支持json格式。

    7. `jsonpaths`

        当导入数据格式为 json 时，可以通过 jsonpaths 指定抽取 Json 数据中的字段。

        `-H "jsonpaths: [\"$.k2\", \"$.k1\"]"`

    8. `strip_outer_array`

        当导入数据格式为 json 时，strip_outer_array 为 true 表示 Json 数据以数组的形式展现，数据中的每一个元素将被视为一行数据。默认值是 false。

        `-H "strip_outer_array: true"`

    9. `json_root`

        当导入数据格式为 json 时，可以通过 json_root 指定 Json 数据的根节点。Doris 将通过 json_root 抽取根节点的元素进行解析。默认为空。

        `-H "json_root: $.RECORDS"`

    10. `send_batch_parallelism`

        整型，用于设置发送批处理数据的并行度，如果并行度的值超过 BE 配置中的 `max_send_batch_parallelism_per_job`，那么作为协调点的 BE 将使用 `max_send_batch_parallelism_per_job` 的值。

    11. `load_to_single_tablet`

        布尔类型，为 true 表示支持一个任务只导入数据到对应分区的一个 tablet，默认值为 false，该参数只允许在对带有 random 分桶的 olap 表导数的时候设置。

    12. `partial_columns` 布尔类型，为 true 表示使用部分列更新，默认值为 false，该参数只允许在表模型为 Unique 且采用 Merge on Write 时设置。一流多表不支持此参数。

    13. `max_filter_ratio`

        采样窗口内，允许的最大过滤率。必须在大于等于0到小于等于1之间。默认值是 1.0。

        采样窗口为 `max_batch_rows * 10`。即如果在采样窗口内，错误行数/总行数大于 `max_filter_ratio`，则会导致例行作业被暂停，需要人工介入检查数据质量问题。

        被 where 条件过滤掉的行不算错误行。

    14. `enclose` When the csv data field contains row delimiters or column delimiters, to prevent accidental truncation, single-byte characters can be specified as brackets for protection. For example, the column separator is ",", the bracket is "'", and the data is "a,'b,c'", then "b,c" will be parsed as a field.

    15. `escape` 转义符。用于转义在csv字段中出现的与包围符相同的字符。例如数据为"a,'b,'c'"，包围符为"'"，希望"b,'c被作为一个字段解析，则需要指定单字节转义符，例如"\"，然后将数据修改为"a,'b,\'c'"。

- `FROM data_source [data_source_properties]`

    数据源的类型。当前支持：

    ```text
    FROM KAFKA
    (
        "key1" = "val1",
        "key2" = "val2"
    )
    ```

    `data_source_properties` 支持如下数据源属性：

    1. `kafka_broker_list`

        Kafka 的 broker 连接信息。格式为 ip:host。多个broker之间以逗号分隔。

        `"kafka_broker_list" = "broker1:9092,broker2:9092"`

    2. `kafka_topic`

        指定要订阅的 Kafka 的 topic。

        `"kafka_topic" = "my_topic"`

    3. `kafka_partitions/kafka_offsets`

        指定需要订阅的 kafka partition，以及对应的每个 partition 的起始 offset。如果指定时间，则会从大于等于该时间的最近一个 offset 处开始消费。

        offset 可以指定从大于等于 0 的具体 offset，或者：

        - `OFFSET_BEGINNING`: 从有数据的位置开始订阅。
        - `OFFSET_END`: 从末尾开始订阅。
        - 时间格式，如："2021-05-22 11:00:00"

        如果没有指定，则默认从 `OFFSET_END` 开始订阅 topic 下的所有 partition。

        ```text
        "kafka_partitions" = "0,1,2,3",
        "kafka_offsets" = "101,0,OFFSET_BEGINNING,OFFSET_END"
        ```

        ```text
        "kafka_partitions" = "0,1,2,3",
        "kafka_offsets" = "2021-05-22 11:00:00,2021-05-22 11:00:00,2021-05-22 11:00:00"
        ```

        注意，时间格式不能和 OFFSET 格式混用。

    4. `property`

        指定自定义kafka参数。功能等同于kafka shell中 "--property" 参数。

        当参数的 value 为一个文件时，需要在 value 前加上关键词："FILE:"。

        关于如何创建文件，请参阅 [CREATE FILE](https://doris.apache.org/zh-CN/docs/sql-manual/Data-Definition-Statements/Create/CREATE-FILE) 命令文档。

        更多支持的自定义参数，请参阅 librdkafka 的官方 CONFIGURATION 文档中，client 端的配置项。如：

        ```text
        "property.client.id" = "12345",
        "property.ssl.ca.location" = "FILE:ca.pem"
        ```

        1. 使用 SSL 连接 Kafka 时，需要指定以下参数：

            ```text
            "property.security.protocol" = "ssl",
            "property.ssl.ca.location" = "FILE:ca.pem",
            "property.ssl.certificate.location" = "FILE:client.pem",
            "property.ssl.key.location" = "FILE:client.key",
            "property.ssl.key.password" = "abcdefg"
            ```

            其中：

            `property.security.protocol` 和 `property.ssl.ca.location` 为必须，用于指明连接方式为 SSL，以及 CA 证书的位置。

            如果 Kafka server 端开启了 client 认证，则还需设置：

            ```text
            "property.ssl.certificate.location"
            "property.ssl.key.location"
            "property.ssl.key.password"
            ```

            分别用于指定 client 的 public key，private key 以及 private key 的密码。

        2. 指定kafka partition的默认起始offset

            如果没有指定 `kafka_partitions/kafka_offsets`，默认消费所有分区。

            此时可以指定 `kafka_default_offsets` 指定起始 offset。默认为 `OFFSET_END`，即从末尾开始订阅。

            示例：

            ```text
            "property.kafka_default_offsets" = "OFFSET_BEGINNING"
            ```

- comment

- 例行导入任务的注释信息。

##### 4.1.3.4、导入示例

**建表**

~~~sql
create table example_db.student_routine_load_kafka
(
    id int,
    name varchar(50),
    age int,
    course_id int,
    score decimal(10,4)
)
DUPLICATE KEY(id)
DISTRIBUTED BY HASH(id) BUCKETS 10;
~~~

**启动kafa生产者并发送指定格式的数据**

~~~shell
[hadoop@hadoop102 data]$ 
[hadoop@hadoop102 data]$ kafka-console-producer.sh --topic doris_topic --bootstrap-server hadoop102:9092,hadoop103:9092,hadoop104:9092
>
~~~

**创建导入任务**

~~~sql
create routine load example_db.student_routine_load_kafka on student_routine_load_kafka
COLUMNS TERMINATED BY ",",
columns(id,name,age,course_id,score)
PROPERTIES
(
    "desired_concurrent_number"="3",
    "max_batch_interval" = "20",
    "max_batch_rows" = "300000",
    "max_batch_size" = "209715200",
    "strict_mode" = "false"
)
FROM KAFKA
(
    "kafka_broker_list" = "hadoop102:9092,hadoop103:9092,hadoop104:9092",
    "kafka_topic" = "doris_topic",
    "property.group.id" = "doris_test_group_01",
    "property.kafka_default_offsets" = "OFFSET_BEGINNING"
);
~~~

**查看表数据**

~~~sql
select * from example_db.student_routine_load_kafka;
~~~

##### 4.1.3.5、查看作业状态

查看**作业**状态的具体命令和示例可以通过 `HELP SHOW ROUTINE LOAD;` 命令查看，具体可参阅[SHOW ROUTINE LOAD](https://doris.apache.org/zh-CN/docs/sql-manual/sql-reference/Show-Statements/SHOW-ROUTINE-LOAD)。

查看**任务**运行状态的具体命令和示例可以通过 `HELP SHOW ROUTINE LOAD TASK;` 命令查看，具体可参阅[SHOW ROUTINE LOAD TASK](https://doris.apache.org/zh-CN/docs/sql-manual/sql-reference/Show-Statements/SHOW-ROUTINE-LOAD-TASK)。

只能查看当前正在运行中的任务，已结束和未开始的任务无法查看。

##### 4.1.3.6、修改作业属性

用户可以修改已经创建的作业。具体说明可以通过 `HELP ALTER ROUTINE LOAD;` 命令查看或参阅 [ALTER ROUTINE LOAD](https://doris.apache.org/zh-CN/docs/sql-manual/sql-reference/Data-Manipulation-Statements/Load/ALTER-ROUTINE-LOAD)。

##### 4.1.3.7、作业控制

用户可以通过 `STOP/PAUSE/RESUME` 三个命令来控制作业的停止，暂停和重启。可以通过 `HELP STOP ROUTINE LOAD;` `HELP PAUSE ROUTINE LOAD;` 以及 `HELP RESUME ROUTINE LOAD;` 三个命令查看帮助和示例，具体命令可参阅 [STOP/PAUSE/RESUME](https://doris.apache.org/zh-CN/docs/sql-manual/sql-reference/Data-Manipulation-Statements/Load/STOP-ROUTINE-LOAD)。

##### 4.1.3.8、其他说明

1. 例行导入作业和 ALTER TABLE 操作的关系

    - 例行导入不会阻塞 SCHEMA CHANGE 和 ROLLUP 操作。但是注意如果 SCHEMA CHANGE 完成后，列映射关系无法匹配，则会导致作业的错误数据激增，最终导致作业暂停。建议通过在例行导入作业中显式指定列映射关系，以及通过增加 Nullable 列或带 Default 值的列来减少这类问题。
    - 删除表的 Partition 可能会导致导入数据无法找到对应的 Partition，作业进入暂停。

2. 例行导入作业和其他导入作业的关系（LOAD, DELETE, INSERT）

    - 例行导入和其他 LOAD 作业以及 INSERT 操作没有冲突。
    - 当执行 DELETE 操作时，对应表分区不能有任何正在执行的导入任务。所以在执行 DELETE 操作前，可能需要先暂停例行导入作业，并等待已下发的 task 全部完成后，才可以执行 DELETE。

3. 例行导入作业和 DROP DATABASE/TABLE 操作的关系

    当例行导入对应的 database 或 table 被删除后，作业会自动 CANCEL。

4. kafka 类型的例行导入作业和 kafka topic 的关系

    当用户在创建例行导入声明的 `kafka_topic` 在kafka集群中不存在时。

    - 如果用户 kafka 集群的 broker 设置了 `auto.create.topics.enable = true`，则 `kafka_topic` 会先被自动创建，自动创建的 partition 个数是由**用户方的kafka集群**中的 broker 配置 `num.partitions` 决定的。例行作业会正常的不断读取该 topic 的数据。
    - 如果用户 kafka 集群的 broker 设置了 `auto.create.topics.enable = false`, 则 topic 不会被自动创建，例行作业会在没有读取任何数据之前就被暂停，状态为 `PAUSED`。

    所以，如果用户希望当 kafka topic 不存在的时候，被例行作业自动创建的话，只需要将**用户方的kafka集群**中的 broker 设置 `auto.create.topics.enable = true` 即可。

5. 在网络隔离的环境中可能出现的问题 在有些环境中存在网段和域名解析的隔离措施，所以需要注意

    1. 创建Routine load 任务中指定的 Broker list 必须能够被Doris服务访问
    2. Kafka 中如果配置了`advertised.listeners`, `advertised.listeners` 中的地址必须能够被Doris服务访问

6. 关于指定消费的 Partition 和 Offset

    Doris 支持指定 Partition 和 Offset 开始消费。新版中还支持了指定时间点进行消费的功能。这里说明下对应参数的配置关系。

    有三个相关参数：

    - `kafka_partitions`：指定待消费的 partition 列表，如："0, 1, 2, 3"。
    - `kafka_offsets`：指定每个分区的起始offset，必须和 `kafka_partitions` 列表个数对应。如："1000, 1000, 2000, 2000"
    - `property.kafka_default_offset`：指定分区默认的起始offset。

    在创建导入作业时，这三个参数可以有以下组合：

    | 组合 | `kafka_partitions` | `kafka_offsets` | `property.kafka_default_offset` | 行为                                                         |
    | ---- | ------------------ | --------------- | ------------------------------- | ------------------------------------------------------------ |
    | 1    | No                 | No              | No                              | 系统会自动查找topic对应的所有分区并从 OFFSET_END 开始消费    |
    | 2    | No                 | No              | Yes                             | 系统会自动查找topic对应的所有分区并从 default offset 指定的位置开始消费 |
    | 3    | Yes                | No              | No                              | 系统会从指定分区的 OFFSET_END 开始消费                       |
    | 4    | Yes                | Yes             | No                              | 系统会从指定分区的指定offset 处开始消费                      |
    | 5    | Yes                | No              | Yes                             | 系统会从指定分区，default offset 指定的位置开始消费          |

7. STOP和PAUSE的区别

    FE会自动定期清理STOP状态的ROUTINE LOAD，而PAUSE状态的则可以再次被恢复启用。

#### 4.1.4、Binlog Load

Binlog Load提供了一种使Doris增量同步用户在Mysql数据库的对数据更新操作的CDC(Change Data Capture)功能。

##### 4.1.4.1、适用场景

- INSERT/UPDATE/DELETE支持
- 过滤Query
- 暂不兼容DDL语句

##### 4.1.4.2、基本原理

当前版本设计中，Binlog Load需要依赖canal作为中间媒介，让canal伪造成一个从节点去获取Mysql主节点上的Binlog并解析，再由Doris去获取Canal上解析好的数据，主要涉及Mysql端、Canal端以及Doris端，总体数据流向如下：

```text
+---------------------------------------------+
|                    Mysql                    |
+----------------------+----------------------+
                       | Binlog
+----------------------v----------------------+
|                 Canal Server                |
+-------------------+-----^-------------------+
               Get  |     |  Ack
+-------------------|-----|-------------------+
| FE                |     |                   |
| +-----------------|-----|----------------+  |
| | Sync Job        |     |                |  |
| |    +------------v-----+-----------+    |  |
| |    | Canal Client                 |    |  |
| |    |   +-----------------------+  |    |  |
| |    |   |       Receiver        |  |    |  |
| |    |   +-----------------------+  |    |  |
| |    |   +-----------------------+  |    |  |
| |    |   |       Consumer        |  |    |  |
| |    |   +-----------------------+  |    |  |
| |    +------------------------------+    |  |
| +----+---------------+--------------+----+  |
|      |               |              |       |
| +----v-----+   +-----v----+   +-----v----+  |
| | Channel1 |   | Channel2 |   | Channel3 |  |
| | [Table1] |   | [Table2] |   | [Table3] |  |
| +----+-----+   +-----+----+   +-----+----+  |
|      |               |              |       |
|   +--|-------+   +---|------+   +---|------+|
|  +---v------+|  +----v-----+|  +----v-----+||
| +----------+|+ +----------+|+ +----------+|+|
| |   Task   |+  |   Task   |+  |   Task   |+ |
| +----------+   +----------+   +----------+  |
+----------------------+----------------------+
     |                 |                  |
+----v-----------------v------------------v---+
|                 Coordinator                 |
|                     BE                      |
+----+-----------------+------------------+---+
     |                 |                  |
+----v---+         +---v----+        +----v---+
|   BE   |         |   BE   |        |   BE   |
+--------+         +--------+        +--------+
```



如上图，用户向FE提交一个数据同步作业。

FE会为每个数据同步作业启动一个canal client，来向canal server端订阅并获取数据。

client中的receiver将负责通过Get命令接收数据，每获取到一个数据batch，都会由consumer根据对应表分发到不同的channel，每个channel都会为此数据batch产生一个发送数据的子任务Task。

在FE上，一个Task是channel向BE发送数据的子任务，里面包含分发到当前channel的同一个batch的数据。

channel控制着单个表事务的开始、提交、终止。一个事务周期内，一般会从consumer获取到多个batch的数据，因此会产生多个向BE发送数据的子任务Task，在提交事务成功前，这些Task不会实际生效。

满足一定条件时（比如超过一定时间、达到提交最大数据大小），consumer将会阻塞并通知各个channel提交事务。

当且仅当所有channel都提交成功，才会通过Ack命令通知canal并继续获取并消费数据。

如果有任意channel提交失败，将会重新从上一次消费成功的位置获取数据并再次提交（已提交成功的channel不会再次提交以保证幂等性）。

整个数据同步作业中，FE通过以上流程不断的从canal获取数据并提交到BE，来完成数据同步。

具体可以查看[Binlog Load - Apache Doris](https://doris.apache.org/zh-CN/docs/data-operate/import/import-way/binlog-load-manual?_highlight=binlog)

#### 4.1.5、Spark Load

Spark Load 通过外部的 Spark 资源实现对导入数据的预处理，提高 Doris 大数据量的导入性能并且节省 Doris 集群的计算资源。主要用于初次迁移，大数据量导入 Doris 的场景。

Spark Load 是利用了 Spark 集群的资源对要导入的数据的进行了排序，Doris BE 直接写文件，这样能大大降低 Doris 集群的资源使用，对于历史海量数据迁移降低 Doris 集群资源使用及负载有很好的效果。

如果用户在没有 Spark 集群这种资源的情况下，又想方便、快速的完成外部存储历史数据的迁移，可以使用 [Broker Load](https://doris.apache.org/zh-CN/docs/sql-manual/sql-reference/Data-Manipulation-Statements/Load/BROKER-LOAD) 。相对 Spark Load 导入，Broker Load 对 Doris 集群的资源占用会更高。

Spark Load 是一种异步导入方式，用户需要通过 MySQL 协议创建 Spark 类型导入任务，并通过 `SHOW LOAD` 查看导入结果。

##### 4.1.5.1、适用场景

- 源数据在 Spark 可以访问的存储系统中，如 HDFS。
- 数据量在 几十 GB 到 TB 级别。

##### 4.1.5.2、基本原理

用户通过 MySQL 客户端提交 Spark 类型导入任务，FE 记录元数据并返回用户提交成功。

Spark Load 任务的执行主要分为以下 5 个阶段。

1. FE 调度提交 ETL 任务到 Spark 集群执行。
2. Spark 集群执行 ETL 完成对导入数据的预处理，包括全局字典构建（ Bitmap 类型）、分区、排序、聚合等。
3. ETL 任务完成后，FE 获取预处理过的每个分片的数据路径，并调度相关的 BE 执行 Push 任务。
4. BE 通过 Broker 读取数据，转化为 Doris 底层存储格式。
5. FE 调度生效版本，完成导入任务。

```text
                 +
                 | 0. User create spark load job
            +----v----+
            |   FE    |---------------------------------+
            +----+----+                                 |
                 | 3. FE send push tasks                |
                 | 5. FE publish version                |
    +------------+------------+                         |
    |            |            |                         |
+---v---+    +---v---+    +---v---+                     |
|  BE   |    |  BE   |    |  BE   |                     |1. FE submit Spark ETL job
+---^---+    +---^---+    +---^---+                     |
    |4. BE push with broker   |                         |
+---+---+    +---+---+    +---+---+                     |
|Broker |    |Broker |    |Broker |                     |
+---^---+    +---^---+    +---^---+                     |
    |            |            |                         |
+---+------------+------------+---+ 2.ETL +-------------v---------------+
|               HDFS              +------->       Spark cluster         |
|                                 <-------+                             |
+---------------------------------+       +-----------------------------+
```

##### 4.1.5.3、全局字典

**适用场景**

目前 Doris 中 Bitmap 列是使用类库 `Roaringbitmap` 实现的，而 `Roaringbitmap` 的输入数据类型只能是整型，因此如果要在导入流程中实现对于 Bitmap 列的预计算，那么就需要将输入数据的类型转换成整型。

在 Doris 现有的导入流程中，全局字典的数据结构是基于 Hive 表实现的，保存了原始值到编码值的映射。

**构建流程**

1. 读取上游数据源的数据，生成一张 Hive 临时表，记为 `hive_table`。
2. 从 `hive_table `中抽取待去重字段的去重值，生成一张新的 Hive 表，记为 `distinct_value_table`。
3. 新建一张全局字典表，记为 `dict_table` ，一列为原始值，一列为编码后的值。
4. 将 `distinct_value_table` 与 `dict_table` 做 Left Join，计算出新增的去重值集合，然后对这个集合使用窗口函数进行编码，此时去重列原始值就多了一列编码后的值，最后将这两列的数据写回 `dict_table`。
5. 将 `dict_table `与 `hive_table` 进行 Join，完成 `hive_table` 中原始值替换成整型编码值的工作。
6. `hive_table `会被下一步数据预处理的流程所读取，经过计算后导入到 Doris 中。

##### 4.1.5.4、数据预处理（DPP）

1. 从数据源读取数据，上游数据源可以是 HDFS 文件，也可以是 Hive 表。
2. 对读取到的数据进行字段映射，表达式计算以及根据分区信息生成分桶字段 `bucket_id`。
3. 根据 Doris 表的 Rollup 元数据生成 RollupTree。
4. 遍历 RollupTree，进行分层的聚合操作，下一个层级的 Rollup 可以由上一个层的 Rollup 计算得来。
5. 每次完成聚合计算后，会对数据根据 `bucket_id `进行分桶然后写入 HDFS 中。
6. 后续 Broker 会拉取 HDFS 中的文件然后导入 Doris Be 中。

##### 4.1.5.5、Hive Bitmap UDF

Spark 支持将 Hive 生成的 Bitmap 数据直接导入到 Doris。详见 [hive-bitmap-udf 文档](https://doris.apache.org/zh-CN/docs/ecosystem/hive-bitmap-udf)

##### 4.1.5.6、基本操作

###### 4.1.5.6.1、配置 ETL 集群

Spark 作为一种外部计算资源在 Doris 中用来完成 ETL 工作，未来可能还有其他的外部资源会加入到 Doris 中使用，如 Spark/GPU 用于查询，HDFS/S3 用于外部存储，MapReduce 用于 ETL 等，因此我们引入 Resource Management 来管理 Doris 使用的这些外部资源。

提交 Spark 导入任务之前，需要配置执行 ETL 任务的 Spark 集群。

~~~sql
-- create spark resource
CREATE EXTERNAL RESOURCE spark_etl
PROPERTIES
(
  type = spark,
  spark_conf_key = spark_conf_value,
  working_dir = path,
  broker = hdfs_broker,
  broker.property_key = property_value,
  broker.hadoop.security.authentication = kerberos,
  broker.kerberos_principal = doris@YOUR.COM,
  broker.kerberos_keytab = /home/doris/my.keytab
  broker.kerberos_keytab_content = ASDOWHDLAWIDJHWLDKSALDJSDIWALD
)

-- drop spark resource
DROP RESOURCE resource_name

-- show resources
SHOW RESOURCES
SHOW PROC "/resources"

-- privileges
GRANT USAGE_PRIV ON RESOURCE resource_name TO user_identity
GRANT USAGE_PRIV ON RESOURCE resource_name TO ROLE role_name

REVOKE USAGE_PRIV ON RESOURCE resource_name FROM user_identity
REVOKE USAGE_PRIV ON RESOURCE resource_name FROM ROLE role_name
~~~



## BUG

1、为FE进行扩容时，新增的FE节点第一次执行时无法从正在运行的节点获取元数据

2、查看Routine Load任务的子任务情况，若查看的job正在运行，会抛出`ERROR 1105 (HY000): NullPointerException, msg: java.lang.NullPointerException: null`错误，将当前任务暂停后会查看子任务情况，命令能够正常通过
