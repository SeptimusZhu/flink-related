## Get Started With Mesos Marathon

Based on CentOS 7

[TOC]

### 准备

#### rpm安装包准备

[zookeeper](http://dl.marmotte.net/rpms/redhat/el7/x86_64/zookeeper-3.4.9-3.el7/zookeeper-3.4.9-3.el7.noarch.rpm)

[mesos](http://repos.mesosphere.com/el/7/x86_64/RPMS/mesos-1.1.0-2.0.107.centos701406.x86_64.rpm)

[marathon](http://repos.mesosphere.com/el-testing/6/x86_64/RPMS/marathon-1.4.0-1.0.560.el6.x86_64.rpm)

#### 环境准备

三台机器的集群，网络互通，关闭防火墙和SELinux

```shell
systemctl stop firewalld && setenforce 0
```

|  节点  |       IP       |                   部署服务                   |
| :--: | :------------: | :--------------------------------------: |
|  1   | 10.120.177.85  | zookeeper、mesos-master、mesos-slave、marathon |
|  2   | 10.120.181.94  | zookeeper、mesos-master、mesos-slave、marathon |
|  3   | 10.120.180.209 | zookeeper、mesos-master、mesos-slave、marathon |

#### 安装

node1、node2、node3:

```shell
rpm -ivh zookeeper*.rpm
yum install libevent libevent-devel -y
rpm -ivh mesos*.rpm
rpm -ivh marathon*.rpm
```

### 配置

#### zookeeper配置

node1:

```shell
echo 1 > /var/lib/zookeeper/myid
```

node2:

```shell
echo 2 > /var/lib/zookeeper/myid
```

node3:

```shell
echo 3 > /var/lib/zookeeper/myid
```

node1、node2、node3配置`/etc/zookeeper/zoo.cfg`

```
server.1=10.120.177.85:2888:3888
server.2=10.120.181.94:2888:3888
server.3=10.120.180.209:2888:3888
```

#### mesos配置

node1、node2、node3配置`/etc/mesos/zk`

```
zk://10.120.177.85:2181,10.120.181.94:2181,10.120.180.209:2181/mesos
```

node1、node2、node3:

```shell
#集群中节点数为3
echo 2 > /etc/mesos-master/quorum
```

#### marathon配置

node1、node2、node3:

```shell
#用于生成marathon.jar
marathon
```

### 启动

node1、node2、node3:

```shell
systemctl restart zookeeper
systemctl restart mesos-master
systemctl restart mesos-slave
marathon run_jar --master zk://10.120.177.85:2181,10.120.181.94:2181,10.120.180.209:2181/mesos --zk zk://10.120.177.85:2181,10.120.181.94:2181,10.120.180.209:2181/marathon
```

可通过三个节点IP访问mesos webUI和marathon webUI，会重定向到zk选举出来的Master节点，以10.120.181.94为例:

```
mesos webUI address: 10.120.181.94:5050
marathon webUI address: 10.120.181.94:8080
```

#### 部署Flink

编写json格式的marathon应用文件描述文件flink-example.json:

```json
{
  "id": "flink",
  "cmd": "/home/flink-1.2.0/bin/mesos-appmaster.sh -Dmesos.master=zk://10.120.177.85:2181,10.120.181.94:2181,10.120.180.209:2181/mesos -Dmesos.initial-tasks=3 -Dmesos.resourcemanager.tasks.cpus=1.0 -Dmesos.resourcemanager.tasks.mem=1024",
  "cpus": 1,
  "mem": 1024,
  "disk": 2048,
  "instances": 1
}
```

通过Marathon WebUI提交或者使用curl命令提交:

```shell
curl -X POST http://10.120.181.94:8080/v2/apps -d @flink-example.json -H "Content-type: application/json"
```



