---
title: Datahub元数据管理系统部署步骤
date: 2022-06-10 17:51:52
tags:
- Datahub
categories:
- Datahub
---

yum install python3 -y

yum install libffi-devel -y
yum install zlib* -y

pip3 install toml

- 安装依赖包

```kotlin
sudo yum install -y yum-utils device-mapper-persistent-data lvm2 
```

- 设置阿里云镜像源



```csharp
sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```



- 安装 Docker-CE

```undefined
sudo yum install docker-ce
```

```bash
# 开机自启
sudo systemctl enable docker 
# 启动docker服务  
sudo systemctl start docker
```

- 添加docker用户组（可选）



```bash
# 1. 建立 Docker 用户组
sudo groupadd docker
# 2.添加当前用户到 docker 组
sudo usermod -aG docker $USER
```

- 镜像加速配置



```bash
# 加速器地址 ：
# 阿里云控制台搜索容器镜像服务
# 进入容器镜像服务， 左侧最下方容器镜像服务中复制加速器地址
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["你的加速器地址"]
}
EOF
# 重启docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```







```
pip3 install docker-compose
```

```
python3 -m pip install --upgrade pip wheel setuptools
python3 -m pip uninstall datahub acryl-datahub || true  # sanity check - ok if it fails
python3 -m pip install --upgrade acryl-datahub
datahub version
```

```
datahub docker quickstart
```

```
datahub docker ingest-sample-data
```

```
datahub docker nuke
```

```
datahub docker nuke --keep-data
datahub docker quickstart
```

使用 datahub docker quickstart 是会报错的，如下图所示，我去问了官方大佬，原因是因为他会去这个网站找个文件，这个网站在中国被锁了，访问不了，所以只能克隆下来整个项目，然后进入目录里执行。 执行完就起来了

```
datahub docker quickstart --quickstart-compose-file ./docker/quickstart/docker-compose-without-neo4j.quickstart.yml
```

注意：

pip install 'acryl-datahub[hive]' 报错

```
yum -y install gcc-g++
// 然后再装这个，不然执行摄入会报错，装就完事了，反正也占不了多少空间
yum install python3-devel
// 装完别急  还有这个
yum install cyrus-sasl-devel
// 别问，问就是摄入失败我花了一两个小时找的解决办法。或者你先直接摄入试试，没有报错你当我放了个屁
```

