---
title: 向往的开发环境搭建（一）：RocketMQ Cluster
date: 2022-08-25 16:40:47
cover: https://api.lattice.vip/manage/file/图片/category-20220826040937428.jpg
tags: [Linux, RocketMQ]
categories: Development Environment
---
## 向往的开发环境搭建（一）：RocketMQ Cluster

> 本文基于Docker虚拟化容器技术，实现一台虚拟机作为宿主机，安装Docker环境部署RocketMQ（2M-2S-Namesrv）集群环境

### 前言

- 前文回顾
    - [I5-12600KF | 升级老式台式机](https://v2ex.com/t/870873)
    - [装机后续：i7-12700 + 华硕 B660M 重炮手装机实操](https://v2ex.com/t/871352)
- 继台式机组装完成后，便开始在脑子中有一个想法：搭建一个程序员必备 /向往的开发环境，最终的蓝图是希望能最大化利用台式机的性能，开机后即可拥有一套现成的本地“云”开发集群环境
- 第一篇便从最近学习的 RocketMQ 出发，基于 Docker 虚拟化容器技术，实现一台虚拟机作为宿主机，安装 Docker 环境部署 RocketMQ （ 2M-2S-Namesrv ）集群环境
- 后续可能会考虑搭建 MySQL 主从、Redis 等中间件集群环境，用以分享学习交流~

### 虚拟机环境

- 官网下载 Centos7 ISO 镜像文件 【CentOS-7-x86_64-Minimal-2207-02】 [清华大学CentOS-7镜像站](https://mirrors.tuna.tsinghua.edu.cn/centos/7.9.2009/isos/x86_64/)

- 启动VamwarePro安装Centos7

- 配置虚拟机参数

    - 4核/16G/80G-SSD
    - IP: 192.168.2.246

- 静态IP配置，vim /etc/sysconfig/network-scripts/ifcfg-ens33

    - BOOTPROTO="static"

    - IPADDR=192.168.2.246

    - NETMASK=255.255.255.0

    - GATEWAY=192.168.2.1

    - DNS1=119.29.29.29

    - ```properties
    TYPE="Ethernet"
    PROXY_METHOD="none"
    BROWSER_ONLY="no"
    BOOTPROTO="static"
    DEFROUTE="yes"
    IPV4_FAILURE_FATAL="no"
    IPV6INIT="yes"
    IPV6_AUTOCONF="yes"
    IPV6_DEFROUTE="yes"
    IPV6_FAILURE_FATAL="no"
    IPV6_ADDR_GEN_MODE="stable-privacy"
    NAME="ens33"
    UUID="deb20895-5574-4f53-89da-d5ccb3a68bf2"
    DEVICE="ens33"
    ONBOOT="yes"
    IPV6_PRIVACY="no"
    IPADDR=192.168.2.246
    NETMASK=255.255.255.0
    GATEWAY=192.168.2.1
    DNS1=119.29.29.29
    ```


### 安装Docker、Docker-Compose

```bash
# 安装docker
[root@localhost ~]# yum install -y yum-utils device-mapper-persistent-data lvm2
[root@localhost ~]# yum -y install docker-ce
[root@localhost ~]# systemctl start docker
[root@localhost ~]# systemctl enable docker
# 安装docker-compose
[root@localhost ~]# yum install docker-compose 
# 查看docker-compose版本信息
[root@localhost ~]# docker-compose -v
```

### 双主双从(2M-2S-ASYNC)

> 异步复制性能最佳，但是存在节点故障时，Master无法正常写入Slave导致部分消息丢失

- 部署方式：`Docker`，共享宿主机配置

- 集群配置：`2M-2S-ASYNC`、`2Namesrv`

#### 宿主机创建配置文件，日志目录，用来映射容器中的路径

```bash
# 创建日志目录文件夹
[root@localhost conf]# mkdir -p /data/docker/rocketmq/{logs-namesrv-m,logs-namesrv-s,logs-broker-a,store-a,conf,logs-broker-b,store-b,logs-broker-a-s,store-a-s,logs-broker-b-s,store-b-s}
# 编写主从配置文件
[root@localhost conf]# vim broker-a.conf
[root@localhost conf]# vim broker-a-s.conf
[root@localhost conf]# vim broker-b.conf
[root@localhost conf]# vim broker-b-s.conf
```

**broker-a.conf**

```properties
#所属集群名字，同一个集群名字相同
brokerClusterName=rocketmq-cluster
#broker名字
brokerName=broker-a
#0表示master >0 表示slave
brokerId=0
#删除文件的时间点，凌晨4点
deleteWhen=04
#文件保留时间 默认是48小时
fileReservedTime=168
#异步复制Master
brokerRole=ASYNC_MASTER
#刷盘方式，ASYNC_FLUSH=异步刷盘，SYNC_FLUSH=同步刷盘 
flushDiskType=ASYNC_FLUSH
#Broker 对外服务的监听端口
listenPort=10911
#nameServer地址，这里nameserver是单台，如果nameserver是多台集群的话，就用分号分割（即namesrvAddr=ip1:port1;ip2:port2;ip3:port3）
namesrvAddr=192.168.2.246:9876
#每个topic对应队列的数量，默认为4，实际应参考consumer实例的数量，值过小不利于consumer负载均衡
defaultTopicQueueNums=8
#是否允许 Broker 自动创建Topic，生产建议关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，生产建议关闭
autoCreateSubscriptionGroup=true
#设置BrokerIP
brokerIP1=192.168.2.246
```

**broker-a-s.conf**

```properties
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样  例如：在a.properties 文件中写 broker-a  在b.properties 文件中写 broker-b
brokerName=broker-a
#0 表示 Master，>0 表示 Slave
brokerId=1
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=168
#Broker 的角色，ASYNC_MASTER=异步复制Master，SYNC_MASTER=同步双写Master，SLAVE=slave节点
brokerRole=SLAVE
#刷盘方式，ASYNC_FLUSH=异步刷盘，SYNC_FLUSH=同步刷盘 
flushDiskType=SYNC_FLUSH
#Broker 对外服务的监听端口
listenPort=12911
#nameServer地址，这里nameserver是单台，如果nameserver是多台集群的话，就用分号分割（即namesrvAddr=ip1:port1;ip2:port2;ip3:port3）
namesrvAddr=192.168.2.246:9876
#每个topic对应队列的数量，默认为4，实际应参考consumer实例的数量，值过小不利于consumer负载均衡
defaultTopicQueueNums=8
#是否允许 Broker 自动创建Topic，生产建议关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，生产建议关闭
autoCreateSubscriptionGroup=true
#设置BrokerIP
brokerIP1=192.168.2.246

```

**broker-b.conf**

```properties
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样  例如：在a.properties 文件中写 broker-a  在b.properties 文件中写 broker-b
brokerName=broker-b
#0 表示 Master，>0 表示 Slave
brokerId=0
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=168
#Broker 的角色，ASYNC_MASTER=异步复制Master，SYNC_MASTER=同步双写Master，SLAVE=slave节点
brokerRole=ASYNC_MASTER
#刷盘方式，ASYNC_FLUSH=异步刷盘，SYNC_FLUSH=同步刷盘 
flushDiskType=SYNC_FLUSH
#Broker 对外服务的监听端口
listenPort=11911
#nameServer地址，这里nameserver是单台，如果nameserver是多台集群的话，就用分号分割（即namesrvAddr=ip1:port1;ip2:port2;ip3:port3）
namesrvAddr=192.168.2.246:9876
#每个topic对应队列的数量，默认为4，实际应参考consumer实例的数量，值过小不利于consumer负载均衡
defaultTopicQueueNums=8
#是否允许 Broker 自动创建Topic，生产建议关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，生产建议关闭
autoCreateSubscriptionGroup=true
#设置BrokerIP
brokerIP1=192.168.2.246
```

**broker-b-s.conf**

```properties
#所属集群名字
brokerClusterName=rocketmq-cluster
brokerName=broker-b
#slave
brokerId=1
deleteWhen=04
fileReservedTime=168
brokerRole=SLAVE
flushDiskType=ASYNC_FLUSH
#Broker 对外服务的监听端口
listenPort=13911
#nameServer地址，这里nameserver是单台，如果nameserver是多台集群的话，就用分号分割（即namesrvAddr=ip1:port1;ip2:port2;ip3:port3）
namesrvAddr=192.168.2.246:9876
#每个topic对应队列的数量，默认为4，实际应参考consumer实例的数量，值过小不利于consumer负载均衡
defaultTopicQueueNums=8
#是否允许 Broker 自动创建Topic，生产建议关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，生产建议关闭
autoCreateSubscriptionGroup=true
#设置BrokerIP
brokerIP1=192.168.2.246
```

#### 构建docker-compose.yml配置文件

```yaml
version: '2'
services:
  namesrv1:
    image: foxiswho/rocketmq:4.8.0
    container_name: namesrv1
    ports:
      - 9876:9876
    volumes:
      - /data/docker/rocketmq/logs-namesrv-m:/home/rocketmq/logs
    environment:
      JAVA_OPT_EXT: -server -Xms256M -Xmx256M -Xmn128m
    command: sh mqnamesrv
  broker-a-m:
      image: foxiswho/rocketmq:4.8.0
      container_name: broker-a-master
      ports:
        - 10909:10909
        - 10911:10911
        - 10912:10912
      volumes:
      - /data/docker/rocketmq/logs-broker-a:/home/rocketmq/logs
      - /data/docker/rocketmq/store-a:/home/rocketmq/store
      - /data/docker/rocketmq/conf/broker-a.conf:/home/rocketmq/rocketmq-4.8.0/conf/broker.conf
      environment:
        JAVA_OPT_EXT: -server -Xms256m -Xmx256m -Xmn128m
        NAMESRV_ADDR: 192.168.2.246:9876
      command: sh mqbroker  -c /home/rocketmq/rocketmq-4.8.0/conf/broker.conf
  broker-b-m:
    image: foxiswho/rocketmq:4.8.0
    container_name: broker-b-master
    ports:
      - 11909:11909
      - 11911:11911
      - 11912:11912
    volumes:
    - /data/docker/rocketmq/logs-broker-b:/home/rocketmq/logs
    - /data/docker/rocketmq/store-b:/home/rocketmq/store
    - /data/docker/rocketmq/conf/broker-b.conf:/home/rocketmq/rocketmq-4.8.0/conf/broker.conf
    environment:
      JAVA_OPT_EXT: -server -Xms256m -Xmx256m -Xmn128m
      NAMESRV_ADDR: 192.168.2.246:9876
    command: sh mqbroker  -c /home/rocketmq/rocketmq-4.8.0/conf/broker.conf
  broker-a-s:
    image: foxiswho/rocketmq:4.8.0
    container_name: broker-a-slave
    ports:
      - 12909:12909
      - 12911:12911
      - 12912:12912
    volumes:
      - /data/docker/rocketmq/logs-broker-a-s:/home/rocketmq/logs
      - /data/docker/rocketmq/store-a-s:/home/rocketmq/store
      - /data/docker/rocketmq/conf/broker-a-s.conf:/home/rocketmq/rocketmq-4.8.0/conf/broker.conf
    environment:
      JAVA_OPT_EXT: -server -Xms256m -Xmx256m -Xmn128m
      NAMESRV_ADDR: 192.168.2.246:9876
    command: sh mqbroker  -c /home/rocketmq/rocketmq-4.8.0/conf/broker.conf
  broker-b-s:
    image: foxiswho/rocketmq:4.8.0
    container_name: broker-b-slave
    ports:
      - 13909:13909
      - 13911:13911
      - 13912:13912
    volumes:
      - /data/docker/rocketmq/logs-broker-b-s:/home/rocketmq/logs
      - /data/docker/rocketmq/store-b-s:/home/rocketmq/store
      - /data/docker/rocketmq/conf/broker-b-s.conf:/home/rocketmq/rocketmq-4.8.0/conf/broker.conf
    environment:
      JAVA_OPT_EXT: -server -Xms256m -Xmx256m -Xmn128m
      NAMESRV_ADDR: 192.168.2.246:9876
    command: sh mqbroker  -c /home/rocketmq/rocketmq-4.8.0/conf/broker.conf
    depends_on:
      - namesrv1
```

#### 主从节点配置信息

| System Info  | IP:PORT               | BrokerId |
| ------------ | --------------------- | -------- |
| `namesrv1`   | 192.168.2.246:`9876`  |          |
| `broker-a`   | 192.168.2.246:`10911` | 0        |
| `broker-a-s` | 192.168.2.246:`12911` | 1        |
| `broker-b`   | 192.168.2.246:`11911` | 0        |
| `broker-b-s` | 192.168.2.246:`13911` | 1        |

#### Docker容器分布

![image-20220820204340646](https://img.lattice.vip/typora/2022/image-20220820204340646.png)

### 部署RocketMQ-Console

获取RocketMQ-Console镜像，连接NameServer中心，新版本以及迁移到新的Repo,名称为RockeMQ-Dashboard

```bash
# 拉取控制台镜像
lattice@lattice-cluster-01:/$docker pull styletang/rocketmq-console-ng
# 启动容器
lattice@lattice-cluster-01:/$docker run -d -e "JAVA_OPTS=-Drocketmq.config.namesrvAddr=192.168.2.246:9876 -Drocketmq.config.isVIPChannel=false" -p 8088:8080 -t styletang/rocketmq-console-ng    
```

部署成功界面

![image-20220819082950746](https://img.lattice.vip/typora/2022/image-20220819082950746.png)

### 遇到的问题

- broker exited 253

    - 最终定位下来问题原因，映射到宿主机的目录卷权限不够，无法应用broker等的配置文件，以及无法操作日志目录

- 宿主机普通用户每次操作docker命令都需要指定sudo，甚是麻烦

    - 解决方案：加入docker组

    - ```bash
    sudo usermod -aG docker xxx
    ```

- 一直报file path的问题，需要手动创建对应目录

    - ```
    mkdir /data/docker/rocketmq/store-a/commitlog/
    mkdir /data/docker/rocketmq/store-a/consumequeue/
    ```

- 搭建集群的过程中确实有遇到一些莫名其妙的问题，例如 Namesrv 最开始部署了两台，9876/9877 ，最终却连不上 9877 ，后来便放弃，使用单节点 Namesrv