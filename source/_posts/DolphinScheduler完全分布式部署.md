---
title: DolphinScheduler完全分布式部署
date: 2022-05-10 09:43:06
tags:
- DolphinScheduler
categories:
- DolphinScheduler
---

# DolphinScheduler-3.0.0-alpha完全分布式部署

## 前置环境准备

- JDK：下载[JDK](https://www.oracle.com/technetwork/java/javase/downloads/index.html) (1.8+)，并将 JAVA_HOME 配置到以及 PATH 变量中。如果你的环境中已存在，可以跳过这步。
- 二进制包：在[下载页面](https://dolphinscheduler.apache.org/zh-cn/download/download.html)下载 DolphinScheduler 二进制包
- 数据库：[PostgreSQL](https://www.postgresql.org/download/) (8.2.15+) 或者 [MySQL](https://dev.mysql.com/downloads/mysql/) (5.7+)，两者任选其一即可，如 MySQL 则需要 JDBC Driver 8.0.16
- 注册中心：[ZooKeeper](https://zookeeper.apache.org/releases.html) (3.4.6+)，[下载地址](https://zookeeper.apache.org/releases.html)
- 进程树分析
  - macOS安装`pstree`
  - Fedora/Red/Hat/CentOS/Ubuntu/Debian安装`psmisc`

> ***注意:\*** DolphinScheduler 本身不依赖 Hadoop、Hive、Spark，但如果你运行的任务需要依赖他们，就需要有对应的环境支持

## 准备 DolphinScheduler 启动环境

### 配置用户免密及权限

创建部署用户，并且一定要配置 `sudo` 免密。以创建 dolphinscheduler 用户为例

```shell
# 创建用户需使用 root 登录
useradd dolphinscheduler

# 添加密码
echo "dolphinscheduler" | passwd --stdin dolphinscheduler

# 配置 sudo 免密
sed -i '$adolphinscheduler  ALL=(ALL)  NOPASSWD: NOPASSWD: ALL' /etc/sudoers
sed -i 's/Defaults    requirett/#Defaults    requirett/g' /etc/sudoers
```

> **注意:**
>
> - 因为任务执行服务是以 `sudo -u {linux-user}` 切换不同 linux 用户的方式来实现多租户运行作业，所以部署用户需要有 sudo 权限，而且是免密的。
> - 如果发现 `/etc/sudoers` 文件中有 "Defaults requirett" 这行，也请注释掉
> - 三台服务器都要进行同样的上述操作

### 配置机器SSH免密登陆

由于安装的时候需要向不同机器发送资源，所以要求**各台机器间能实现SSH免密登陆**。配置免密登陆的步骤如下

```shell
# 登陆bigdata1
[root@bigdata1 dolphinscheduler]# su dolphinscheduler
[root@bigdata1 dolphinscheduler]# cd /home/dolphinscheduler/.ssh
[root@bigdata1 dolphinscheduler]# ssh-keygen -t rsa
# 然后敲（三个回车），就会生成两个文件id_rsa（私钥）、id_rsa.pub（公钥）
[root@bigdata1 dolphinscheduler]# ssh-copy-id bigdata1
[root@bigdata1 dolphinscheduler]# ssh-copy-id bigdata2
[root@bigdata1 dolphinscheduler]# ssh-copy-id bigdata3
```

> **注意：** 三台服务器都要进行同样的上述操作

### 启动zookeeper

进入 zookeeper 的安装目录，将 `zoo_sample.cfg` 配置文件复制到 `conf/zoo.cfg`，并将 `conf/zoo.cfg` 中 dataDir 中的值改成 `dataDir=./tmp/zookeeper`

```shell
# 启动 zookeeper
./bin/zkServer.sh start
```

> **注意：**若系统已经存在zookeeper则忽略此步骤。

## 修改相关配置

登陆部署账户dolphinscheduler，将下载的部署包放在 **/home/dolphinscheduler/**  对应账户的家目录下,

```sh
[dolphinscheduler@bigdata1 dolphinscheduler]$ pwd
/home/dolphinscheduler/apache-dolphinscheduler-3.0.0-alpha-bin
[dolphinscheduler@bigdata1 dolphinscheduler]$ mv apache-dolphinscheduler-3.0.0-alpha-bin dolphinscheduler
[dolphinscheduler@bigdata1 dolphinscheduler]$ cd dolphinscheduler
# 修改目录权限，使得部署用户对二进制包解压后的 apache-dolphinscheduler-*-bin 目录有操作权限
[dolphinscheduler@bigdata1 dolphinscheduler]$ chown -R dolphinscheduler:dolphinscheduler dolphinscheduler
```


完成基础环境的准备后，需要根据你的机器环境修改配置文件。配置文件可以在目录 `<bin|alert-server|api-server|master-server|worker-server|tools>/dolphinscheduler_env.sh|application.yaml` 六个sh文件和5个yaml文件以及 `bin/env/install_env.sh` 找到他们。

### 修改 `install_env.sh` 文件

文件 `install_env.sh` 描述了哪些机器将被安装 DolphinScheduler 以及每台机器对应安装哪些服务。您可以在路径 `bin/env/install_env.sh` 中找到此文件，配置详情如下。

```shell
# ---------------------------------------------------------
# INSTALL MACHINE
# ---------------------------------------------------------
# A comma separated list of machine hostname or IP would be installed DolphinScheduler,
# including master, worker, api, alert. If you want to deploy in pseudo-distributed
# mode, just write a pseudo-distributed hostname
# Example for hostnames: ips="ds1,ds2,ds3,ds4,ds5", Example for IPs: ips="192.168.8.1,192.168.8.2,192.168.8.3,192.168.8.4,192.168.8.5"
ips=${ips:-"bigdata1,bigdata2,bigdata3"}

# Port of SSH protocol, default value is 22. For now we only support same port in all `ips` machine
# modify it if you use different ssh port
sshPort=${sshPort:-"22"}

# A comma separated list of machine hostname or IP would be installed Master server, it
# must be a subset of configuration `ips`.
# Example for hostnames: masters="ds1,ds2", Example for IPs: masters="192.168.8.1,192.168.8.2"
masters=${masters:-"bigdata1"}

# A comma separated list of machine <hostname>:<workerGroup> or <IP>:<workerGroup>.All hostname or IP must be a
# subset of configuration `ips`, And workerGroup have default value as `default`, but we recommend you declare behind the hosts
# Example for hostnames: workers="ds1:default,ds2:default,ds3:default", Example for IPs: workers="192.168.8.1:default,192.168.8.2:default,192.168.8.3:default"
workers=${workers:-"bigdata2:default,bigdata3:default"}

# A comma separated list of machine hostname or IP would be installed Alert server, it
# must be a subset of configuration `ips`.
# Example for hostname: alertServer="ds3", Example for IP: alertServer="192.168.8.3"
alertServer=${alertServer:-"bigdata3"}

# A comma separated list of machine hostname or IP would be installed API server, it
# must be a subset of configuration `ips`.
# Example for hostname: apiServers="ds1", Example for IP: apiServers="192.168.8.1"
apiServers=${apiServers:-"bigdata1"}

# The directory to install DolphinScheduler for all machine we config above. It will automatically be created by `install.sh` script if not exists.
# Do not set this configuration same as the current path (pwd)
installPath=${installPath:-"/opt/module/dolphinscheduler"}

# The user to deploy DolphinScheduler for all machine we config above. For now user must create by yourself before running `install.sh`
# script. The user needs to have sudo privileges and permissions to operate hdfs. If hdfs is enabled than the root directory needs
# to be created by this user
deployUser=${deployUser:-"dolphinscheduler"}

# The root of zookeeper, for now DolphinScheduler default registry server is zookeeper.
zkRoot=${zkRoot:-"/dolphinscheduler"}
```

### 修改 `dolphinscheduler_env.sh` 文件

**修改所有配置文件**在 `<bin|alert-server|api-server|master-server|worker-server|tools>/dolphinscheduler_env.sh` 可以找到。此描述了下列配置：

- DolphinScheduler 的数据库配置，详细配置方法见[初始化数据库](https://dolphinscheduler.apache.org/zh-cn/docs/latest/user_doc/guide/installation/pseudo-cluster.html#初始化数据库)
- 一些任务类型外部依赖路径或库文件，如 `JAVA_HOME` 和 `SPARK_HOME`都是在这里定义的
- 注册中心`zookeeper`
- 服务端相关配置，比如缓存，时区设置等

如果您不使用某些任务类型，您可以忽略任务外部依赖项，但您必须根据您的环境更改 `JAVA_HOME`、注册中心和数据库相关配置。

```sh
export HADOOP_HOME=${HADOOP_HOME:-/usr/hdp/3.1.4.0-315/hadoop}
export HADOOP_CONF_DIR=${HADOOP_CONF_DIR:-/usr/hdp/3.1.4.0-315/hadoop/etc/hadoop}
#export SPARK_HOME1=${SPARK_HOME1:-/opt/soft/spark1}
export SPARK_HOME2=${SPARK_HOME2:-/usr/hdp/3.1.4.0-315/spark2}
export PYTHON_HOME=${PYTHON_HOME:-/usr/bin/python}
export JAVA_HOME=${JAVA_HOME:-/opt/jdk/jdk1.8.0_181}
export HIVE_HOME=${HIVE_HOME:-/usr/hdp/3.1.4.0-315/hive}
export FLINK_HOME=${FLINK_HOME:-/opt/module/flink}
export DATAX_HOME=${DATAX_HOME:-/opt/module/datax-web/datax}

export PATH=$HADOOP_HOME/bin:$SPARK_HOME1/bin:$SPARK_HOME2/bin:$PYTHON_HOME/bin:$JAVA_HOME/bin:$HIVE_HOME/bin:$FLINK_HOME/bin:$DATAX_HOME/bin:$PATH

export SPRING_JACKSON_TIME_ZONE=${SPRING_JACKSON_TIME_ZONE:-UTC}
export DATABASE=${DATABASE:-mysql}
export SPRING_PROFILES_ACTIVE=${DATABASE}
export SPRING_DATASOURCE_DRIVER_CLASS_NAME=com.mysql.cj.jdbc.Driver
export SPRING_DATASOURCE_URL=jdbc:mysql://bigdata1:3306/dolphinscheduler?useUnicode=true&characterEncoding=UTF-8
export SPRING_DATASOURCE_USERNAME=dolphinscheduler
export SPRING_DATASOURCE_PASSWORD=dolphinscheduler
export SPRING_CACHE_TYPE=${SPRING_CACHE_TYPE:-none}

export MASTER_FETCH_COMMAND_NUM=${MASTER_FETCH_COMMAND_NUM:-10}

export REGISTRY_TYPE=${REGISTRY_TYPE:-zookeeper}
export REGISTRY_ZOOKEEPER_CONNECT_STRING=${REGISTRY_ZOOKEEPER_CONNECT_STRING:-bigdata1,bigdata2,bigdata3:2181}
```

### 修改application.properties文件

**修改所有配置文件**在 alert-server|api-server|master-server|worker-server|tools ->application.properties

```yaml
## mysql配置修改
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://bigdata1:3306/dolphinscheduler
    username: dolphinscheduler
    password: dolphinscheduler
	......
# Override by profile
spring:
  config:
    activate:
      on-profile: mysql
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://bigdata1:3306/dolphinscheduler?useUnicode=true&characterEncoding=UTF-8

## ZK配置修改
registry:
  type: zookeeper
  zookeeper:
    namespace: dolphinscheduler
    connect-string: bigdata1:2181,bigdata2:2181,bigdata3:2181
```

### 修改配置开启资源存储

**修改所有配置文件**在 alert-server|api-server|bin->env|master-server|worker-server|tools   -->  common.properties

```properties
# resource storage type: HDFS, S3, NONE
resource.storage.type=HDFS

# resource store on HDFS/S3 path, resource file will store to this hadoop hdfs path, self configuration, please make sure the directory exists on hdfs and have read write permissions. "/dolphinscheduler" is recommended
resource.upload.path=/dolphinscheduler

# whether to startup kerberos
hadoop.security.authentication.startup.state=false
# java.security.krb5.conf path
java.security.krb5.conf.path=/opt/krb5.conf
# login user from keytab username
login.user.keytab.username=hdfs-mycluster@ESZ.COM
# login user from keytab path
login.user.keytab.path=/opt/hdfs.headless.keytab
# kerberos expire time, the unit is hour
kerberos.expire.time=2

# resource view suffixs
#resource.view.suffixs=txt,log,sh,bat,conf,cfg,py,java,sql,xml,hql,properties,json,yml,yaml,ini,js
# if resource.storage.type=HDFS, the user must have the permission to create directories under the HDFS root path
hdfs.root.user=hdfs
# if resource.storage.type=S3, the value like: s3a://dolphinscheduler; if resource.storage.type=HDFS and namenode HA is enabled, you need to copy core-site.xml and hdfs-site.xml to conf dir
fs.defaultFS=hdfs://bigdata1:8020
```



## 初始化数据库

DolphinScheduler 元数据存储在关系型数据库中，目前支持 PostgreSQL 和 MySQL，如果使用 MySQL 则需要手动下载 [mysql-connector-java 驱动](https://downloads.mysql.com/archives/c-j/) (8.0.16) 并移动到 DolphinScheduler 的 lib目录下（`tools/libs/`）。注意为了更好操作请将 mysql驱动放到alert-server|api-server|master-server|worker-server|tools下面的libs里面下面以 MySQL 为例，说明如何初始化数据库

对于mysql 5.6 / 5.7：

```shell
mysql -uroot -p

mysql> CREATE DATABASE dolphinscheduler DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;

# dolphinscheduler 和 dolphinscheduler 为希望的用户名和密码
mysql> GRANT ALL PRIVILEGES ON dolphinscheduler.* TO 'dolphinscheduler'@'%' IDENTIFIED BY 'dolphinscheduler';
mysql> GRANT ALL PRIVILEGES ON dolphinscheduler.* TO 'dolphinscheduler'@'localhost' IDENTIFIED BY 'dolphinscheduler';

mysql> flush privileges;
```

对于mysql 8：

```shell
mysql -uroot -p

mysql> CREATE DATABASE dolphinscheduler DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;

# dolphinscheduler 和 dolphinscheduler 为希望的用户名和密码
mysql> CREATE USER 'dolphinscheduler'@'%' IDENTIFIED BY 'dolphinscheduler';
mysql> GRANT ALL PRIVILEGES ON dolphinscheduler.* TO 'dolphinscheduler'@'%';
mysql> CREATE USER 'dolphinscheduler'@'localhost' IDENTIFIED BY 'dolphinscheduler';
mysql> GRANT ALL PRIVILEGES ON dolphinscheduler.* TO 'dolphinscheduler'@'localhost';
mysql> FLUSH PRIVILEGES;
```

然后修改`./bin/dolphinscheduler_env.sh`，将username和password改成你在上一步中设置的用户名{user}和密码{password}

```shell
export DATABASE=${DATABASE:-mysql}
export SPRING_PROFILES_ACTIVE=${DATABASE}
export SPRING_DATASOURCE_DRIVER_CLASS_NAME=com.mysql.cj.jdbc.Driver
export SPRING_DATASOURCE_URL=jdbc:mysql://bigdata1:3306/dolphinscheduler?useUnicode=true&characterEncoding=UTF-8
export SPRING_DATASOURCE_USERNAME=dolphinscheduler
export SPRING_DATASOURCE_PASSWORD=dolphinscheduler
```

完成上述步骤后，您已经为 DolphinScheduler 创建一个新数据库，现在你可以通过快速的 Shell 脚本来初始化数据库

```shell
sh tools/bin/create-schema.sh
```

## 启动 DolphinScheduler

登陆上面创建的系统相应的**部署用户**运行以下命令自动完成分布式部署、运行、并启动，部署后的运行日志将存放在 logs 文件夹内

```shell
[root@bigdata1 dolphinscheduler]# su dolphinscheduler
[dolphinscheduler@bigdata1 dolphinscheduler]$ sh ./bin/install.sh
```

> ***注意:\*** 第一次部署的话，可能出现 5 次`sh: bin/dolphinscheduler-daemon.sh: No such file or directory`相关信息，次为非重要信息直接忽略即可

## 登录 DolphinScheduler

浏览器访问地址 http://localhost:12345/dolphinscheduler 即可登录系统UI。默认的用户名和密码是 **admin/dolphinscheduler123**

## 启停服务

```shell
# 登陆系统账户dolphinscheduler
[root@bigdata1 dolphinscheduler]# su dolphinscheduler
# 一键停止集群所有服务
sh ./bin/stop-all.sh

# 一键开启集群所有服务
sh ./bin/start-all.sh

# 启停 Master
sh ./bin/dolphinscheduler-daemon.sh stop master-server
sh ./bin/dolphinscheduler-daemon.sh start master-server

# 启停 Worker
sh ./bin/dolphinscheduler-daemon.sh start worker-server
sh ./bin/dolphinscheduler-daemon.sh stop worker-server

# 启停 Api
sh ./bin/dolphinscheduler-daemon.sh start api-server
sh ./bin/dolphinscheduler-daemon.sh stop api-server

# 启停 Alert
sh ./bin/dolphinscheduler-daemon.sh start alert-server
sh ./bin/dolphinscheduler-daemon.sh stop alert-server
```

> **注意：**Python gateway service 默认与 api-server 一起启动，如果您不想启动 Python gateway service 请通过更改 api-server 配置文件 `api-server/conf/application.yaml` 中的 `python-gateway.enabled : false` 来禁用它。

