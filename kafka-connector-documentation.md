## Kafka Connector Documentation 

> 诸国栋 2017.1.13
>
> 基于Apache Flink 1.3-SNAPSHOT

[TOC]

## overview

针对不同的kafka客户端版本，Flink connector也有与之对应的三个版本，分别适配Kafka 0.8.2.2、0.9.0.1和0.10.0.1版本。

## 0.8.x version

#### architecture

  ![](http://code.huawei.com/real-time-team/roadmap/raw/1a6e47e10f52800fa94b62a070e77ab1611c6a53/pictures/08kafka_connector.PNG)

​	Kafka消费者类`FlinkKafkaConsumer08`和Kafka生产者类`FlinkKafkaProducer08`与用户在DataStream API中自定义的算子函数类似，都实现了`Function`接口。这两个类基本上是对老版本的接口做封装，多数方法在对应的父类Base类中实现，其中consumer是source端，而producer是sink端。

​	在Kafka的consumer类中创建了`Kafka08Fetcher`类，在fetcher类方法中创建`SimpleConsumerThread`线程类，该守护线程循环调用Kafka接口`SimpleConsumer.fetch`用于接收Kafka partition发来的数据，并调用`Kafka08Fetcher`的基类`AbstractFetcher`的`emitRecord`方法传递数据给`StreamSource`算子。

​	Kafka的producer类`FlinkKafkaProducer08`的父类`FlinkKafkaProducerBase`实现了接口`SinkFunction`的`invoke`方法，当数据到来，对数据进行序列化后，调用kafka client的`send`方法，找到一个partition并发送。

#### consumer application sample

- dependencies

```xml
    <dependencies>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-scala_2.10</artifactId>
            <version>1.3-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-kafka-0.8_2.10</artifactId>
            <version>1.3-SNAPSHOT</version>
        </dependency>
    </dependencies>
```

- main source

```java
public class ReadFromKafka {
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
}
```


#### consumer related API

```java
  FlinkKafkaConsumer08(String topic, DeserializationSchema<T> valueDeserializer, Properties props)
  FlinkKafkaConsumer08(List<String> topics, DeserializationSchema<T> deserializer, Properties props)
  FlinkKafkaConsumer08(String topic, KeyedDeserializationSchema<T> deserializer, Properties props)
  FlinkKafkaConsumer08(List<String> topics, KeyedDeserializationSchema<T> deserializer, Properties props)
  FlinkKafkaConsumer081(String topic, DeserializationSchema<T> valueDeserializer, Properties props)
  FlinkKafkaConsumer082(String topic, DeserializationSchema<T> valueDeserializer, Properties props)
```

  接口说明：适配Kafka 0.8.x版本的Flink消费者源节点自定义函数构造函数，包含单/多topic、是否为key/value数据等。不建议使用`FlinkKafkaConsumer081`和`FlinkKafkaConsumer082`接口，所有以上的构造函数最终调用构造函数`FlinkKafkaConsumer08(List<String> topics, KeyedDeserializationSchema<T> deserializer, Properties props)`，非key/value pair数据的情况下`deserializer`中实现的接口方法中只对value进行反序列化。

  参数说明：

* `topic`：Kafka topic名

* `valueDserializer`：反序列化模型，需要实现反序列化模型接口`DeserializationSchema<T>`，并重写反序列化接口`T deserialize(byte[] message) `，实现对从Kafka接收的字节流的反序列化

* `props`：Kafka消费者端运行参数，主要包括fetcher和offset handler相关配置项，具体如下表所示：



|             配置项名              |                    含义                    |                    示例                    |
| :---------------------------: | :--------------------------------------: | :--------------------------------------: |
|             topic             |               Kafka topic名               |               --topic test               |
|       bootstrap.servers       |    Kafka集群连接串，可以由多个host:port组成，以逗号分隔     |    --bootstrap.servers localhost:9092    |
|       zookeeper.connect       |              Zookeeper地址端口               |    --zookeeper.connect localhost:2181    |
|           group.id            | Consumer的Group id，同一个group下的多个Consumer不会pull到重复的消息，不同group下的Consumer则会保证pull到每一条消息，同一个group下的consumer不能多于partition |            --group.id myGroup            |
|      session.timeout.ms       | 会话超时时间，如果kafka coordinator在超时时间内没有收到来自消费者的心跳请求，将rebalance整个group，并认为consumer已经dead |       -- session.timeout.ms  6000        |
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
|       metric.reporters        | metrics reporter类列表，需要实现`MetricReporter`接口 |                                          |
|       key.deserializer        | key/value格式消息中key的反序列化类名，需要实现`Deserializer`接口 |        --key.deserializer classA         |
|      value.deserializer       | key/value格式消息中value的反序列化类名，需要实现`Deserializer`接口 |       --value.deserializer classB        |
|     flink.disable-metrics     |      Flink私有配置，当设为true时，关闭metrics统计      |       --flink.disable-metrics true       |

#### producer application sample

* pom dependencies

  same as consumer

* main source

```java
//a valid parameter sample: --topic test --bootstrap.servers localhost:9092
public class WriteIntoKafka {
	public static void main(String[] args) throws Exception {
		// create execution environment
		StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

		// parse user parameters
		ParameterTool parameterTool = ParameterTool.fromArgs(args);

		// add a simple source which is writing some strings
		DataStream<String> messageStream = env.addSource(new SimpleStringGenerator());

		// write stream to Kafka
		messageStream.addSink(new FlinkKafkaProducer08<>(parameterTool.getRequired("bootstrap.servers"),
				parameterTool.getRequired("topic"),
				new SimpleStringSchema()));

		env.execute();
	}
	// user-defined source function, continously generating string data
	public static class SimpleStringGenerator implements SourceFunction<String> {
		private static final long serialVersionUID = 2174904787118597072L;
		boolean running = true;
		long i = 0;
		@Override
		public void run(SourceContext<String> ctx) throws Exception {
			while(running) {
				ctx.collect("element-"+ (i++));
				Thread.sleep(10);
			}
		}

		@Override
		public void cancel() {
			running = false;
		}
	}
```



#### producer related API

```java
FlinkKafkaProducer08(String brokerList, String topicId, SerializationSchema<IN> serializationSchema)
FlinkKafkaProducer08(String topicId, SerializationSchema<IN> serializationSchema, Properties producerConfig)
FlinkKafkaProducer08(String topicId, SerializationSchema<IN> serializationSchema, Properties producerConfig, KafkaPartitioner<IN> customPartitioner)
FlinkKafkaProducer08(String brokerList, String topicId, KeyedSerializationSchema<IN> serializationSchema)
FlinkKafkaProducer08(String topicId, KeyedSerializationSchema<IN> serializationSchema, Properties producerConfig)
FlinkKafkaProducer08(String topicId, KeyedSerializationSchema<IN> serializationSchema, Properties producerConfig, KafkaPartitioner<IN> customPartitioner)
```

接口说明：与consumer类似，所有的构造函数最终调用最下面的构造函数`FlinkKafkaProducer08(String topicId, KeyedSerializationSchema<IN> serializationSchema, Properties producerConfig, KafkaPartitioner<IN> customPartitioner)` ，需要传入的参数包括broker的ip列表（最终会转换成`producerConfig` ，producer所需的配置只有topic名和broker ip这两项）、topic名、序列化类和用户自定义的partition选择器。

## 0.9.x version

## 0.10.x version

#### architecture

![](http://code.huawei.com/real-time-team/roadmap/raw/1a6e47e10f52800fa94b62a070e77ab1611c6a53/pictures/10kafka_connector.PNG)

由图可见，10版本的consumer和fetcher只是对09版本的一层接口适配，09版本的fetcher在实现上也跟08版本有差异，fetcher主线程`Kafka09Fetcher`创建接收线程`KafkaConsumerThread`，两者持有`Handover`这个中间类的相同实例，接收线程调用kafka client的`poll`接口从Kafka服务器接收数据，传递给`Handover`后由fetcher线程从`Handover`拿到数据后，调用`AbstractFetcher`的`emitRecord`方法传递数据给`StreamSource`算子。

10版本的producer是function和operator的混合体，可以使用08版本的API调用方式，通过`DataStream.addSink`添加，也可以调用`FlinkKafkaProducer010.writeToKafkaWithTimestamps()`方法，支持将Kafka事件携带的Event Time写入Kafka（以DataStream API方式创建的producer，只实现了`invoke(IN value)`接口，在该框架下无法获取事件类，而通过后者方法创建的producer，重写`StreamSink`的`processElement(StreamRecord element)`方法，可以获取`StreamRecord`，从而获取事件携带的时间戳，详见producer application sample和API描述中不同的使用方式举例说明）。在数据处理流程上，跟08版本保持一致。

#### consumer application sample

* dependencies

  ```xml
      <dependencies>
          <dependency>
              <groupId>org.apache.flink</groupId>
              <artifactId>flink-streaming-scala_2.10</artifactId>
              <version>1.3-SNAPSHOT</version>
          </dependency>
          <dependency>
              <groupId>org.apache.flink</groupId>
              <artifactId>flink-connector-kafka-0.10_2.10</artifactId>
              <version>1.3-SNAPSHOT</version>
          </dependency>
      </dependencies>
  ```

  ​

* main source

  与08版本基本相同

  ```java
  public class ReadFromKafka {
  //a valid parameter sample: --topic test --bootstrap.servers localhost:9092 --zookeeper.connect localhost:2181 --group.id myGroup
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);

      ParameterTool parameterTool = ParameterTool.fromArgs(args);
      DataStream<String> messageStream = env.addSource(new FlinkKafkaConsumer010<>(
          parameterTool.getRequired("topic"), new SimpleStringSchema(),
          parameterTool.getProperties()));

      messageStream.rebalance().map(value -> "Kafka and Flink says: " + value).print();

      env.execute();
    }
  }
  ```

  ​

#### consumer related API

```java
FlinkKafkaConsumer010(String topic, DeserializationSchema<T> valueDeserializer, Properties props)
FlinkKafkaConsumer010(String topic, KeyedDeserializationSchema<T> deserializer, Properties props)
FlinkKafkaConsumer010(List<String> topics, DeserializationSchema<T> deserializer, Properties props)
FlinkKafkaConsumer010(List<String> topics, KeyedDeserializationSchema<T> deserializer, Properties props)
```

接口说明：参考[08版本接口说明](#consumer-related-api)。

参数说明：props参数与08版本的有所不同，以下具体列举说明，其他参数含义相同，参考08版本参数说明。

* consumer和producer共有配置

  | 配置项                      | 含义                                       |
  | ------------------------ | ---------------------------------------- |
  | bootstrap.servers        | Kafka集群ip/port列表，以"host1:port1,host2:port2,..."的形式，无需将整个kafka集群列全 |
  | metadata.max.age.ms      | 强制刷新元数据的周期时间，无论是否有新的broker或者partition    |
  | send.buffer.bytes        | 发送数据时TCP发送缓冲区(SO_SNDBUF)大小               |
  | receive.buffer.bytes     | 接收数据时TCP接收缓冲区(SO_RCVBUF)大小               |
  | client.id                | 向Kafka服务器发送请求时的客户端标识，便于在服务端日志中区分         |
  | reconnect.backoff.ms     | 重连服务器的时间间隔，用于防止频繁重连                      |
  | retry.backoff.ms         | 向topic partition发送请求返回失败后的等待重试的时间，用于防止频繁请求 |
  | metrics.sample.window.ms | metrics中的sample清理周期，当窗口周期到来，清除计数结果并重新开始计数 |
  | metrics.num.samples      | metrics包含的sample数量                       |
  | metric.reporters         | metrics reporter类列表，需要实现`MetricReporter`接口 |
  | security.protocol        | 安全相关，与broker通信时使用的协议，默认为PLAINTEXT，可选SSL和SASL_SSL，具体配置在另外文档中描述 |
  | connections.max.idle.ms  | 连接的空闲等待时间，到达后连接关闭                        |
  | request.timeout.ms       | Kafka客户端发送请求后等待服务端响应的超时时间                |

* consumer配置

  | 配置项名                  | 含义                                       |
  | --------------------- | ---------------------------------------- |
  | group.id              | 参见08版本                                   |
  | max.poll.records      | 单次`poll`调用返回的最大记录数                       |
  | session.timeout.ms    | 参见08版本，取值范围在group.min.session.timeout.ms 和group.max.session.timeout.ms之间 |
  | heartbeat.interval.ms | 心跳间隔，向Kafka coordinator发送的心跳消息用于检测consumer的会话是否处于active状态，以及在consumer加入或者离开consumer group时及时触发re-balance机制，应当小于session.timeout.ms的值，大于其1/3。 |
  |                       |                                          |
  |                       |                                          |
  |                       |                                          |
  |                       |                                          |
  |                       |                                          |

####  producer application sample

* pom dependencies

  same as consumer

* main source

  ```java
  //a valid parameter sample: --topic test --bootstrap.servers localhost:9092
  public class WriteIntoKafka {
      public static void main(String[] args) throws Exception {
          // create execution environment
          StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

          // parse user parameters
          ParameterTool parameterTool = ParameterTool.fromArgs(args);

          // add a simple source which is writing some strings
          DataStream<String> messageStream = env.addSource(new SimpleStringGenerator());

          // 使用原有的DataStream API方式
          messageStream.addSink(new FlinkKafkaProducer010<>(parameterTool.getRequired("bootstrap.servers"),
                  parameterTool.getRequired("topic"),
                  new SimpleStringSchema()));
          // 使用StreamSink方式，用于处理事件携带的时间戳
  //        FlinkKafkaProducer010.writeToKafkaWithTimestamps(messageStream,
  //                parameterTool.getRequired("topic"),
  //                new SimpleStringSchema(),
  //                parameterTool.getProperties());

          env.execute();
      }

      // user-defined source function, continously generating string data
      public static class SimpleStringGenerator implements SourceFunction<String> {
          private static final long serialVersionUID = 2174904787118597072L;
          boolean running = true;
          long i = 0;

          @Override
          public void run(SourceContext<String> ctx) throws Exception {
              while (running) {
                  ctx.collect("element-" + (i++));
                  Thread.sleep(10);
              }
          }

          @Override
          public void cancel() {
              running = false;
          }
      }
  }
  ```

#### producer related API

* DataStream用法API

```java
FlinkKafkaProducer010(String brokerList, String topicId, SerializationSchema<T> serializationSchema)
FlinkKafkaProducer010(String topicId, SerializationSchema<T> serializationSchema, Properties producerConfig)
FlinkKafkaProducer010(String topicId, SerializationSchema<T> serializationSchema, Properties producerConfig, KafkaPartitioner<T> customPartitioner)
FlinkKafkaProducer010(String brokerList, String topicId, KeyedSerializationSchema<T> serializationSchema)
FlinkKafkaProducer010(String topicId, KeyedSerializationSchema<T> serializationSchema, Properties producerConfig)
FlinkKafkaProducer010(String topicId, KeyedSerializationSchema<T> serializationSchema, Properties producerConfig, KafkaPartitioner<T> customPartitioner)
```

接口说明：参见[08版本](#producer-related-api)。

* StreamSink 用法接口

```java
FlinkKafkaProducer010Configuration<T> writeToKafkaWithTimestamps(DataStream<T> inStream, String topicId, KeyedSerializationSchema<T> serializationSchema, Properties producerConfig)
FlinkKafkaProducer010Configuration<T> writeToKafkaWithTimestamps(DataStream<T> inStream, String topicId, SerializationSchema<T> serializationSchema, Properties producerConfig)
FlinkKafkaProducer010Configuration<T> writeToKafkaWithTimestamps(DataStream<T> inStream, String topicId, KeyedSerializationSchema<T> serializationSchema, Properties producerConfig, KafkaPartitioner<T> customPartitioner)
```

接口说明：对应DataStream API的另一种用法，支持将事件携带的时间写入Kafka，参数含义与DataStream用法相同，接口用法参见[sample](#producer-application-sample-1)。producer配置项参数`producerConfig`包含的所有配置项详见下表（其中与consumer重叠部分的配置项参见[consumer related API章节](#consumer-related-api-1)，不在下面列举）：

| 配置项                       | 含义                                       |
| ------------------------- | ---------------------------------------- |
| metadata.fetch.timeout.ms | 第一次往topic发送数据前，我们必须获取topic相关的元数据以便得知topic相关的服务器信息和partition信息，该配置项表示获取的超时时间 |
|                           |                                          |
|                           |                                          |
|                           |                                          |
|                           |                                          |
|                           |                                          |
|                           |                                          |



