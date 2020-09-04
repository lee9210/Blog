title: 集群管理工具 KafkaAdminClient —— 原理与示例
date: 2018-01-19
tag: 
categories: Kafka
permalink: Kafka/KafkaAdminClient-1
author: 朱小厮
from_url: https://blog.csdn.net/u013256816/article/details/79996056
wechat_url: 

-------

摘要: 原创出处 https://blog.csdn.net/u013256816/article/details/79996056 「朱小厮」欢迎转载，保留摘要，谢谢！


-------


### 前言

一般情况下，我们都习惯使用Kafka中bin目录下的脚本工具来管理查看Kafka，但是有些时候需要将某些管理查看的功能集成到系统（比如Kafka Manager）中，那么就需要调用一些API来直接操作Kafka了。在Kafka0.11.0.0版本之前，可以通过kafka-core包（Kafka的服务端代码，采用Scala编写）下的AdminClient和AdminUtils来实现部分的集群管理操作，比如笔者之前在[Kafka解析之topic创建(1)](https://blog.csdn.net/u013256816/article/details/79303825)和[Kafka解析之topic创建(2)](https://blog.csdn.net/u013256816/article/details/79303846)两篇文章中所讲解的Topic的创建就用到了AdminUtils类。而在Kafka0.11.0.0版本之后，又多了一个AdminClient，这个是在kafka-client包下的，这是一个抽象类，具体的实现是org.apache.kafka.clients.admin.KafkaAdminClient，这个就是本文所要陈述的重点了。

### 功能与原理介绍

在Kafka官网中这么描述AdminClient：The AdminClient API supports managing and inspecting topics, brokers, acls, and other Kafka objects. 具体的KafkaAdminClient包含了一下几种功能（以Kafka1.0.0版本为准）：

1. 创建Topic：createTopics(Collection<NewTopic> newTopics)
2. 删除Topic：deleteTopics(Collection<String> topics)
3. 罗列所有Topic：listTopics()
4. 查询Topic：describeTopics(Collection<String> topicNames)
5. 查询集群信息：describeCluster()
6. 查询ACL信息：describeAcls(AclBindingFilter filter)
7. 创建ACL信息：createAcls(Collection<AclBinding> acls)
8. 删除ACL信息：deleteAcls(Collection<AclBindingFilter> filters)
9. 查询配置信息：describeConfigs(Collection<ConfigResource> resources)
10. 修改配置信息：alterConfigs(Map<ConfigResource, Config> configs)
11. 修改副本的日志目录：alterReplicaLogDirs(Map<TopicPartitionReplica, String> replicaAssignment)
12. 查询节点的日志目录信息：describeLogDirs(Collection<Integer> brokers)
13. 查询副本的日志目录信息：describeReplicaLogDirs(Collection<TopicPartitionReplica> replicas)
14. 增加分区：createPartitions(Map<String, NewPartitions> newPartitions)

其内部原理是使用Kafka自定义的一套二进制协议来实现，详细可以参见[Kafka协议](https://kafka.apache.org/protocol)。主要实现步骤：

1. 客户端根据方法的调用创建相应的协议请求，比如创建Topic的createTopics方法，其内部就是发送CreateTopicRequest请求。
2. 客户端发送请求至Kafka Broker。
3. Kafka Broker处理相应的请求并回执，比如与CreateTopicRequest对应的是CreateTopicResponse。
4. 客户端接收相应的回执并进行解析处理。
   和协议有关的请求和回执的类基本都在org.apache.kafka.common.requests包中，AbstractRequest和AbstractResponse是这些请求和回执类的两个基本父类。

### 示例

下面就以创建Topic来举一个简单的KafkaAdminClient的使用案例，【代码清单1】：

```java
private static final String NEW_TOPIC = "topic-test2";
private static final String brokerUrl = "localhost:9092";

private static AdminClient adminClient;

@BeforeClass
public static void beforeClass(){
    Properties properties = new Properties();
    properties.put(CommonClientConfigs.BOOTSTRAP_SERVERS_CONFIG, brokerUrl);
    adminClient = AdminClient.create(properties);
}

@AfterClass
public static void afterClass(){
    adminClient.close();
}

@Test
public void createTopics() {
    NewTopic newTopic = new NewTopic(NEW_TOPIC,4, (short) 1);
    Collection<NewTopic> newTopicList = new ArrayList<>();
    newTopicList.add(newTopic);
    adminClient.createTopics(newTopicList);
}
```

示例中的createTopics()方法就创建了一个分区数为4，副本因子为1的“topic-test2”的Topic。

### 代码剖析

下面来详细介绍一下KafkaAdminClient中现有的listTopics()方法(这个方法的实现相对干净利落，代码量少、易于讲解)的实现方式，以便可以了解KafkaAdminClient中的大体脉络。listTopics()方法的具体代码如【代码清单2】所示：

```java
public ListTopicsResult listTopics(final ListTopicsOptions options) {
    final KafkaFutureImpl<Map<String, TopicListing>> topicListingFuture = new KafkaFutureImpl<>();
    final long now = time.milliseconds();
    runnable.call(new Call("listTopics", calcDeadlineMs(now, options.timeoutMs()),
        new LeastLoadedNodeProvider()) {
        @Override
        AbstractRequest.Builder createRequest(int timeoutMs) {
            return MetadataRequest.Builder.allTopics();
        }
        @Override
        void handleResponse(AbstractResponse abstractResponse) {
            MetadataResponse response = (MetadataResponse) abstractResponse;
            Cluster cluster = response.cluster();
            Map<String, TopicListing> topicListing = new HashMap<>();
            for (String topicName : cluster.topics()) {
                boolean internal = cluster.internalTopics().contains(topicName);
                if (!internal || options.shouldListInternal())
                    topicListing.put(topicName, new TopicListing(topicName, internal));
            }
            topicListingFuture.complete(topicListing);
        }
        @Override
        void handleFailure(Throwable throwable) {
            topicListingFuture.completeExceptionally(throwable);
        }
    }, now);
    return new ListTopicsResult(topicListingFuture);
}
```

listTopics()方法接收一个ListTopicsOptions类型的参数，KafkaAdminClient中基本所有的应用类方法都有一个类似XXXOptions类型的参数，这个类型一般只包含timeoutMs这个成员变量，用来设定请求的超时时间，如果没有指定则使用默认的request.timeout.ms参数值，即30000ms。就拿查询Topic信息所对应的DescribeTopicsOptions来说，其就包含一个timeoutMs参数，具体如【代码清单3】所示：

```java
public class DescribeTopicsOptions extends AbstractOptions<DescribeTopicsOptions> {}
public abstract class AbstractOptions<T extends AbstractOptions> {
    private Integer timeoutMs = null;
    @SuppressWarnings("unchecked")
    public T timeoutMs(Integer timeoutMs) {
        this.timeoutMs = timeoutMs;
        return (T) this;
    }
    public Integer timeoutMs() {
        return timeoutMs;
    }
}
```

不过ListTopicsOptions扩展了一个成员变量listInternal，用来指明是否需要罗列内部Topic，比如在[Kafka解析之topic创建(1)](https://blog.csdn.net/u013256816/article/details/79303825)中提及的“__consumer_offsets”和“transaction_state”就是两个内部Topic。ListTopicsOptions的代码如【代码清单4】所示：

```java
public class ListTopicsOptions extends AbstractOptions<ListTopicsOptions> {
    private boolean listInternal = false;
    public ListTopicsOptions listInternal(boolean listInternal) {
        this.listInternal = listInternal;
        return this;
    }
    public boolean shouldListInternal() {
        return listInternal;
    }
}
```

listInternal的值默认为false，如果同时要罗列出目前的内部Topic的话就需要将这个listInternal设置为true，示例代码如【代码清单5】所示：

```java
@Test
public void listTopicsIncludeInternal() throws ExecutionException, InterruptedException {
    ListTopicsOptions listTopicsOptions = new ListTopicsOptions();
    listTopicsOptions.listInternal(true);
    ListTopicsResult result = adminClient.listTopics(listTopicsOptions);
    Collection<TopicListing> list = result.listings().get();
    System.out.println(list);
}
```

接下去继续讲解listTopics()方法，其返回值为ListTopicResult类型。与ListTopicsOptions对应，KafkaAdminClient中基本所有的应用类方法都有一个类似XXXResult类型的返回值，其内部一般包含一个KafkaFuture，用于异步发送请求之后等待操作结果。KafkaFuture实现了Java中的Future接口，用来支持链式调用以及其他异步编程模型，可以看成是Java8中CompletableFuture的一个小型版本，其中也有类似thenApply、complete、completeExceptionally的方法。

再来看代码清单2中的 runnable.call(new Call(“listTopics”, calcDeadlineMs(now, options.timeoutMs()),new LeastLoadedNodeProvider()) 这行代码，runnable的类型是AdminClientRunnable，其是KafkaAdminClient负责处理与服务端交互请求的服务线程。AdminClientRunnable中的call方法用作入队一个Call请求，进而对其处理。Call请求代表与服务端的一次请求交互，比如listTopics和createTopics都是一次Call请求，AdminClientRunnable线程负责处理这些Call请求。

Call类是一个抽象类，构造方法接收三个参数：本次请求的名称callName、超时时间deadlineMs、以及节点提供器nodeProvider。nodeProvider是NodeProvider类型，用来提供本次请求所交互的Broker节点。Call类中还有3个抽象方法：createRequest()、handleResponse()、handleFailure()，分别用来创建请求、处理回执和处理失败。在代码清单2中，对于listTopics(）方法而言，其内部原理就是发送MetadataRequest请求然后处理MetadataResponse，其处理逻辑峰封装在createRequest()、handleResponse()、handleFailure()这三个方法之中了。

综上，如果要自定义实现一个功能，只需要三个步骤：

1. 自定义XXXOptions;
2. 自定义XXXResult返回值；
3. 自定义Call，然后挑选合适的XXXRequest和XXXResponse来实现Call类中的3个抽象方法。

KafkaAdminClient目前而言尚未形成一个完全体，里面还可以扩展很多功能，就拿上一篇文章《[如何获取Kafka的消费者详情——从Scala到Java的切换](https://blog.csdn.net/u013256816/article/details/79968647)》中介绍的而言，目前KafkaAdminClient尚未实现describeConsumerGroup和listGroupOffsets的功能，所以需要进一步的升级改造。篇幅限制，这部分内容将在下一篇文章进行介绍，如果想要先睹为快，可以参考下[代码实现](https://github.com/hiddenzzh/kafka/blob/master/src/main/java/org/apache/kafka/clients/admin/app/KafkaConsumerGroupService.java)，详细的逻辑解析敬请期待….
