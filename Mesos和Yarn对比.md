## Mesos和YARN对比

#### overview

Mesos是一个开源的资源管理系统，可以对集群中的资源做弹性管理，目前twitter, apple等公司在大量使用mesos管理集群资源，国内也有不少公司在使用mesos，比如豆瓣、数人云。

Apache Hadoop 2.0中包含的YARN，是一个通过Hadoop集群提供资源管理的统一平台。

|           | Mesos                                    | YARN                           |
| --------- | ---------------------------------------- | ------------------------------ |
| 实现语言      | C++                                      | Java                           |
| 基本架构      | master/slaves                            | master/slaves                  |
| 对hadoop支持 | 粗粒度支持，hadoop直接以服务形式运行                    | 细粒度支持，每个MR/Spark作业均是一个短应用      |
| 对docker支持 | 中等，主要支持Mesos自有容器                         | 很差                             |
| 社区参与者     | 较少                                       | 较多                             |
| 两者关系      | 对hadoop以及上层计算框架均有支持                      | 资源分配算法采用了mesos中的DRF            |
| 对长服务的支持   | 好，Marathon框架天然支持长服务                      | 一般，YARN主要用于调度批处理任务             |
| 框架担任的角色   | 各个计算框架需在Mesos中部署后才能使用，完全融入到Mesos中        | 计算框架只是作为客户端的库使用，不在YARN中部署即可使用  |
| 调度机制      | 双层调度；第一层由Mesos Master将空闲资源分配给框架，第二层由各个框架自带的调度器对资源的使用进行分配 | 双层调度，第一层由RM分配资源，第二层再对框架的任务进行调度 |
| 资源分配      | 先粗粒度分配，再细粒度分配                            | 细粒度分配                          |
| 资源利用率     | 基于Linux OS级别资源隔离方案(包括cgroup、gpu等)，利用率较高  | 只支持基于cgroup cpu的隔离，利用率较低       |



### 关键差异

#### 1.资源隔离

Mesos支持基于cgroup的大部分资源隔离，包括cpu、memory、disk io等；也支持基于POSIX系统的disk占用隔离，以及nvidia GPU的隔离等。

YARN在Apache Hadoop 2.7.3版本中，只支持基于cgroup cpu的隔离，而对内存的隔离采用了更为灵活的线程监控的方式，避免因内存抖动导致oom，在最新的2.8.0版本中，增加了基于cgroup disk io的隔离[YARN-2619](https://issues.apache.org/jira/browse/YARN-2619)。

总体来说，YARN在资源隔离方面还有很多需要改进的地方，比如支持更细粒度和更多类型的资源隔离。细粒度的资源隔离方式更有利于流服务的最大化资源利用率和计费诉求，以及未来可能存在的图计算对GPU资源的隔离需求。

#### 2.长服务支持

流作业经常以long-running service的状态存在，这会引入以下问题：

* 资源竞争与饿死
* 服务高可用
* 日志滚动收集
* 服务发现
* 资源伸缩



#### 3.社区生态

YARN作为hadoop生态的核心之一，对hadoop上层计算框架的支持很好，包括mapreduce、spark、flink等，而且社区参与人数较多。Apache Twill经过近三年的孵化后，于2016年7月成为Apache顶级项目，增加了YARN的易用性。

Mesos社区因日趋成熟而且没有厂商干预和商业诉求，社区活跃度不高。Mesos对hadoop上层框架也有支持，其中[Mesos-hadoop](https://github.com/mesos/hadoop)项目处于不活跃状态，只支持MapReduce1.0版本，不支持2.0和YARN。Spark的第一个实现就是基于Mesos，后来才有了YARN的实现，2016年的[Spark Survey](http://cdn2.hubspot.net/hubfs/438089/DataBricks_Surveys_-_Content/2016_Spark_Survey/2016_Spark_Infographic.pdf)显示YARN上的Spark部署是Mesos的5倍。

Flink对YARN的集成已经基本完善。Flink在1.2.0版本引入对Mesos的支持，1.2.0发布当时社区用户[调查结果](https://dcos.io/blog/2017/apache-flink-on-dc-os-and-apache-mesos/)显示，30%的用户在Mesos上部署Flink(在YARN上部署数量的一半)。目前1.2.0版本目前已经支持通过脚本启动AM进程(包括RM和JM)与Mesos Master交互， Flink社区会在近期增加对Mesos的集成度，包括在[FLIP-6](https://docs.google.com/document/d/1zwBP3LKnJ0LI1R4-9hLXBVjOXAkQS5jKrGYYoPz-xRk/edit#heading=h.giuwq6q8d23j)中提到的动态资源申请和端到端任务提交。

```
The Flink community is actively working on improvements beyond the 1.2 release, and will add two key components in the near future.
1. Dynamic resource allocation: In Flink 1.2, it’s not possible to dynamically adjust the number of tasks allocated to a job running on Mesos. FLIP-6 will address this issue by separating the concerns of all deployment components. A dispatcher component will receive jobs and spawn Flink clusters, and the new ResourceManager will dynamically allocate new tasks if more resources are needed.
2. Integration with the Flink CLI: In the future, it will be possible to start a Mesos cluster per job using the Flink CLI. Right now, a user must first start a Flink cluster on Mesos and then submit a long-running cluster session.
```

[Apache Myriad](http://myriad.apache.org/)项目由eBay、Mesosphere和MapR共同开发，把YARN作为一个在Mesos上运行的框架，基于YARN的应用和Mesos其他框架在集群中同时运行调度。该项目在2015年交由ASF孵化，至今尚未毕业。





##### references

* https://www.oreilly.com/ideas/a-tale-of-two-clusters-mesos-and-yarn
* https://hortonworks.com/apache/yarn/
* https://biaobean.gitbooks.io/dcos/content/mesos_hadoop.html
* https://github.com/mesos/hadoop
* https://github.com/mesos/spark
* http://dongxicheng.org/mapreduce-nextgen/hadoop-yarn-resource-isolation/
* http://dongxicheng.org/mapreduce-nextgen/yarn-mesos-borg/