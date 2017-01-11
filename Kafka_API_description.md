## Documentation of Kafka Connector In Flink 

> 本文的API等描述基于Apache Flink 1.1.4

### 0.8.x version

* architecture of Kafka connector

  ![diagram](08kafka_connector.PNG)

  ​	Kafka消费者类`FlinkKafkaConsumer08`和Kafka生产者类`FlinkKafkaProducer08`与用户在DataStream API中自定义的算子函数类似，都应用了`Function`接口，其中consumer是source端，而producer是sink端。

  ​	在Kafka的consumer类中定义了`Kafka08Fetcher`类，在该类方法中创建`SimpleConsumerThread`线程类，该守护线程用于接收Kafka partition发来的数据，并调用`Kafka08Fetcher`的基类`AbstractFetcher`的`emitRecord`方法传递数据给`StreamSource`算子。

  ​

* flink consumer application sample

  ```java
  //a valid parameter sample: --topic test --bootstrap.servers localhost:9092 --zookeeper.connect localhost:2181 --group.id myGroup
  public static void main(String[] args) throws Exception {
  	// create execution environment
  	StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
  	env.setParallelism(1);

  	// parse user parameters
  	ParameterTool parameterTool = ParameterTool.fromArgs(args);
  	DataStream<String> messageStream = env.addSource(new FlinkKafkaConsumer08<>(
  	parameterTool.getRequired("topic"), new SimpleStringSchema(), parameterTool.getProperties()));

  	// print() will write the contents of the stream to the TaskManager's standard out stream
  	// the rebelance call is causing a repartitioning of the data so that all machines
  	// see the messages (for example in cases when "num kafka partitions" < "num flink operators"
      // note: need to set java language level to 8 in order to support lamda expression
  	messageStream.rebalance().map(value -> "Kafka and Flink says: " + value).print();

  	env.execute();
  }
  ```

  ​

* consumer related API

  ```java
  FlinkKafkaConsumer08(String topic, DeserializationSchema<T> valueDeserializer, Properties props)
  FlinkKafkaConsumer08(List<String> topics, DeserializationSchema<T> deserializer, Properties props)
  FlinkKafkaConsumer08(String topic, KeyedDeserializationSchema<T> deserializer, Properties props)
  FlinkKafkaConsumer08(List<String> topics, KeyedDeserializationSchema<T> deserializer, Properties props)
  FlinkKafkaConsumer081(String topic, DeserializationSchema<T> valueDeserializer, Properties props)
  FlinkKafkaConsumer082(String topic, DeserializationSchema<T> valueDeserializer, Properties props)
  ```

  接口说明：适配Kafka 0.8.x版本的Flink消费者源节点自定义函数构造函数，包含单/多topic、是否为key/value数据等。不建议使用`FlinkKafkaConsumer081`和`FlinkKafkaConsumer082`接口

  参数说明：

  * `topic`：Kafka topic名

  * `valueDserializer`：反序列化模型，需要应用反序列化模型接口`DeserializationSchema<T>`，并重写反序列化接口`T deserialize(byte[] message) `，实现对从Kafka接收的字节流的反序列化

  * `props`：Kafka消费者端运行参数，主要包括fetcher和offset handler相关配置项，具体如下表所示

    |             配置项名              |                    含义                    |                    示例                    |
    | :---------------------------: | :--------------------------------------: | :--------------------------------------: |
    |             topic             |               Kafka topic名               |               --topic test               |
    |       bootstrap.servers       |    Kafka集群连接串，可以由多个host:port组成，以逗号分隔     |    --bootstrap.servers localhost:9092    |
    |       zookeeper.connect       |              Zookeeper地址端口               |    --zookeeper.connect localhost:2181    |
    |           group.id            | Consumer的Group id，同一个group下的多个Consumer不会pull到重复的消息，不同group下的Consumer则会保证pull到每一条消息，同一个group下的consumer不能多于partition |            --group.id myGroup            |
    |      session.timeout.ms       | 会话超时时间，如果kafka coordinator在超时时间内没有收到来自消费者的心跳请求，将触发rebalance并认为consumer已经dead |       -- session.timeout.ms  6000        |
    |      heartbeat.frequency      | consumer每session.timeout.ms/heartbeat.frequency向coordinator发送一次心跳并等待返回 |         -- heartbeat.frequency 5         |
    |      enable.auto.commit       | 使能周期性地告知kafka当前已处理的消息offset，周期为auto.commit.interval.ms |        --enable.auto.commit true         |
    |    auto.commit.interval.ms    |                    见上                    |      --auto.commit.interval.ms 1000      |
    | partition.assignment.strategy | 分配策略，用于指定线程消费那些分区的消息，默认采用range策略（按照阶段平均分配）。比如分区有10个、线程数有3个，则线程 1消费0,1,2,3，线程2消费4,5,6,线程3消费7,8,9。另外一种是roundrobin(循环分配策略)，官方文档中写有使用该策略有两个前提条件的，所以一般不要去设定。 | --partition.assignment.strategy roundrobin |
    |       auto.offset.reset       | 指定从哪个offset开始消费消息，默认为largest，即从最新的消息开始消费，consumer只能得到其启动后producer产生的消息；也可配成smallest，则从最早的消息开始 |       --auto.offset.reset smallest       |
    |        fetch.min.bytes        | server发送到consumer的最小数据，如果不满足这个数值则会等待知道满足指定大小。 |           --fetch.min.bytes 1            |
    |       fetch.max.wait.ms       | 当Kafka服务器收集到fetch.min.bytes大小的数据之前，无法及时响应fetch请求的超时时间 |         --fetch.max.wait.ms 6000         |
    |   metadata.fetch.timeout.ms   | 获取topic相关元数据超时时间，超时情况下consumer报`TimeoutException`异常 |     --metadata.fetch.timeout.ms 6000     |
    |      total.memory.bytes       | consumer最大缓存大小，当consumer订阅了多个topic时，所有跟partition的连接共享该缓存大小 |        --total.memory.bytes 8192         |
    |      fetch.buffer.bytes       | 一次fetch的内存大小，该配置应当大于服务器一条消息的最大长度，否则consumer可能在fetch时卡住 |        --fetch.buffer.bytes 4096         |
    |           client.id           | 向Kafka服务器发送请求时携带的consumer相关字符串，用于定义一个ip/port之外的逻辑应用名 |      --client.id my_flink_consumer       |
    |  socket.receive.buffer.bytes  |              socket接收缓冲区大小               |   --socket.receive.buffer.bytes 65536    |
    |     reconnect.backoff.ms      |      consumer重连broker的时间间隔，用于防止频繁重连      |        --reconnect.backoff.ms 128        |
    |      metrics.num.samples      |            metrics包含的sample数量            |         --metrics.num.samples 2          |
    |   metrics.sample.window.ms    | metrics中的sample清理周期，当窗口周期到来，清除计数结果并重新开始计数 |     --metrics.sample.window.ms 6000      |
    |       metric.reporters        | metrics reporter类列表，需要应用`MetricReporter`接口 |                                          |
    |       key.deserializer        | key/value格式消息中key的反序列化类名，需要应用`Deserializer`接口 |        --key.deserializer classA         |
    |      value.deserializer       | key/value格式消息中value的反序列化类名，需要应用`Deserializer`接口 |       --value.deserializer classB        |
    |     flink.disable-metrics     |      Flink私有配置，当设为true时，关闭metrics统计      |       --flink.disable-metrics true       |

  ​



  ​

### 0.9.x version

### 0.10.x version

> 暂不支持
