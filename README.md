# RocketmqExtendTools

Since the official API provided by RocketMQ and RocketmqConsole only provide some basic functions, in actual development, developers often need to expand based on the basic API. The purpose of this project is to expand its functions based on RocketMQ, provide some easy-to-use APIs for functions that are not directly implemented by the official, for everyone to use or learn from, reduce learning costs, and improve efficiency.

The functions currently implemented are:

1. Message idempotent deduplication solution to prevent repeated consumption of messages.
2. The elastic scaling of Producer and Consumer automatically scales producers and consumers based on resource usage.
3. Realize some common functions: obtain the logical position of the message in the queue, obtain the tps of production and consumption, obtain the corresponding relationship between the queue and its consumers, etc.



Since the current project does not consider distributed and some actual use scenarios, the project will be improved in the future, mainly in the following aspects:

1. The currently used idempotent deduplication scheme is a heavyweight scheme, and the problem of repeated message consumption is a low probability problem. For the processing of small probability problems, heavyweight schemes may not be used. Because there are caches or database operations before and after each message consumption, the first problem is the low efficiency, and the second problem is the consistency of redis and MySQL.

A lightweight solution will be added later. For a system, it is tolerant. For example, repeated messages will only appear within 5 minutes. Then you can directly use redis to prevent repeated consumption. Redis stores message information. The survival time is set to 6 minutes, so as long as the message can be found in redis, the current message is a repeated message, otherwise it is not.

2. The elastic scaling of Producer and Consumer only considers the elastic scaling of the local machine. RocketMQ is a distributed system. Both consumers and producers exist in the form of clusters. The elastic scaling should be designed to communicate through RPC or network. Scale producers and consumers in producer and consumer clusters.
3. Several commonly used functions are implemented through the RocketMQ client in the rocketmq-tools package. Calling the API is actually performing network communication, that is, the obtained information is not real-time information, but the information of the previous snapshot status. Fetching real-time status will be considered later.



## Message idempotent deduplication

The implementation of message idempotent deduplication is in the duplicate package, which supports the use of redis or MySQL as idempotent tables, and also supports the use of redis and MySQL for double checking.



When using message idempotent deduplication, the startup configuration of Consumer is not much different from usual. The only difference is that the user needs to customize a class, let it inherit AvoidDuplicateMsgListener, and register it as the MessageListener of consumer.



### Principle flow chart

![image-20220621212235579](file://D:\Desktop\Principle flow chart.jpg)



### Instructions for use

1. Create a custom class to inherit from AvoidDuplicateMsgListener. Note that the parameter of its constructor is DuplicateConfig, and the constructor of the parent class is called during construction. The doHandleMsg method rewritten by this class is the logic for actually consuming messages.

```java
public class DuplicateTest extends AvoidDuplicateMsgListener {

    // 构造函数必须这么写，调用父类的构造函数，参数为DuplicateConfig
    public DuplicateTest(DuplicateConfig duplicateConfig) {
        super(duplicateConfig);
    }

    // 重写的该方法为消费消息的逻辑
    @Override
    protected boolean doHandleMsg(MessageExt messageExt) {
        switch (messageExt.getTopic()) {
            case "DUPLICATE-TEST-TOPIC": {
                log.info("消费中..., messageId: {}", messageExt.getMsgId());
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                break;
            }
        }

        return true;
    }
}
```

2. Depending on the use of redis or/and MySQL, use the uniqKey of the message or the business key (the business key refers to the key set by calling the setKeys method when sending the message) for deduplication, and call different enableAvoidDuplicateConsumeConfig methods to obtain DuplicateConfig.

```java
// 使用redis和MySQL,指定MySQL表名,applicationName表示针对什么应用做去重，相同的消息在不同应用的去重是隔离处理的,msgKeyStrategy可以为USERKEY_AS_MSG_KEY也可以为UNIQKEY_AS_MSG_KEY
public static DuplicateConfig enableAvoidDuplicateConsumeConfig(String applicationName, int msgKeyStrategy, StringRedisTemplate redisTemplate, JdbcTemplate jdbcTemplate, String tableName) {
    return new DuplicateConfig(applicationName, AVOID_DUPLICATE_ENABLE, msgKeyStrategy, redisTemplate, jdbcTemplate, tableName);
}

// 使用redis
public static DuplicateConfig enableAvoidDuplicateConsumeConfig(String applicationName, int msgKeyStrategy, StringRedisTemplate redisTemplate) {
    return new DuplicateConfig(applicationName, AVOID_DUPLICATE_ENABLE, msgKeyStrategy, redisTemplate);
}

// 使用MySQL
public static DuplicateConfig enableAvoidDuplicateConsumeConfig(String applicationName, int msgKeyStrategy, JdbcTemplate jdbcTemplate, String tableName) {
    return new DuplicateConfig(applicationName, AVOID_DUPLICATE_ENABLE, msgKeyStrategy, jdbcTemplate, tableName);
}
```

Note that the columns of the MySQL idempotent table are fixed, and the table name can be customized. The table creation statement is:

```sql
CREATE TABLE `xxxxx` (
`application_name` varchar(255) NOT NULL COMMENT '消费的应用名（可以用消费者组名称）',
`topic` varchar(255) NOT NULL COMMENT '消息来源的topic（不同topic消息不会认为重复）',
`tag` varchar(16) NOT NULL COMMENT '消息的tag（同一个topic不同的tag，就算去重键一样也不会认为重复），没有tag则存""字符串',
`msg_uniq_key` varchar(255) NOT NULL COMMENT '消息的唯一键（建议使用业务主键）',
`status` varchar(16) NOT NULL COMMENT '这条消息的消费状态',
`expire_time` bigint(20) NOT NULL COMMENT '这个去重记录的过期时间（时间戳）',
UNIQUE KEY `uniq_key` (`application_name`,`topic`,`tag`,`msg_uniq_key`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 ROW_FORMAT=COMPACT;
```

3. New custom class, register it as the MessageListener of the consumer, and idempotent de-duplication will take effect. Here we use redis as an example. In other cases, the code can be found in the MainTest class of the duplicate package under the test package.

```java
@Test
public static void redisTest() throws MQClientException {
    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("DUPLICATE-TEST-GROUP");
    consumer.subscribe("DUPLICATE-TEST-TOPIC", "*");
    // 指定NameServer,需要用户指定NameServer地址
    consumer.setNamesrvAddr("192.168.20.100:9876");

    consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

    String appName = consumer.getConsumerGroup();
    // 需要用户提供StringRedisTemplate
    StringRedisTemplate stringRedisTemplate = xxx;
    // 调用DuplicateConfig的enableAvoidDuplicateConsumeConfig方法创建DuplicateConfig
    DuplicateConfig duplicateConfig = DuplicateConfig.enableAvoidDuplicateConsumeConfig(appName, USERKEY_AS_MSG_KEY, stringRedisTemplate);
    // 创建用户自定义的Listener类
    AvoidDuplicateMsgListener msgListener = new DuplicateTest(duplicateConfig);
    // 将Listener注册到consumer，幂等去重即生效
    consumer.registerMessageListener(msgListener);
    consumer.start();
}
```



## Elastic scaling of Producer and Consumer

The elastic scaling of Producer and Consumer is implemented in the scaling package, which supports optional monitoring of cpu, memory, produce tps, and consume tps indicators, and elastically expands or reduces the number of Producers or Consumers based on this.



### Instructions for use

#### Consumer Elastic Scaling

1. To create a consumer startup template, the user needs to provide the consumer group name, NameServer address, ConsumeFromWhere, subscribed topic and tag, consumption mode (cluster or broadcast), and MessageListenerConcurrently.

```java
Map<String, String> topicAndTag = new HashMap<>();
topicAndTag.put("SCALE", "*");
// 参数分别为：消费者组名称、NameServer地址、ConsumeFromWhere、订阅的topic和tag（用Map保存）、消费模式（集群或广播）、MessageListenerConcurrently
ConsumerStartingTemplate consumerStartingTemplate = new ConsumerStartingTemplate("SCALE_TEST_CONSUMER_GROUP", "192.168.20.100:9876", ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET,
        topicAndTag, MessageModel.CLUSTERING, new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
        // 逐条消费消息
        for (MessageExt msg : msgs) {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        // 返回消费状态:消费成功
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});
```

2. To create a consumer scaling group, the user needs to provide the scaling group name, the minimum number of consumer instances, the maximum number of consumer instances, the cooling time (that is, the shortest time interval between two consecutive scalings), the consumer startup template, and the shrinking strategy (remove The latest or earliest created consumer, ConsumerRemoveStrategy provides getOldestRemoveStrategy() and getNewestRemoveStrategy() methods).

```java
// 参数分别为伸缩组名称、最少消费者实例数、最多消费者实例数、冷却时间（即连续两次伸缩的最短时间间隔，单位为秒）、消费者启动模板、收缩策略（去掉最早或最晚创建的消费者）
ConsumerScalingGroup consumerScalingGroup = new ConsumerScalingGroup("CONSUMER-SCALE", 0, 3, 5, consumerStartingTemplate, new ConsumerRemoveStrategy().getOldestRemoveStrategy());
```

3. Create a monitor class (there are four monitors: CpuMonitorRunnable, MemoryMonitorRunnable, ProduceTpsMonitor, ConsumeTpsMonitor, which monitor cpu, memory, production tps, and consumption tps respectively), requiring the user to provide the monitor name, scaling group name, scaling group, and scheduled execution The cron expression, statistical duration (unit: second), statistical method (maximum, minimum, average value), expansion threshold, shrinking threshold, how many times the threshold is reached consecutively to trigger scaling. In addition, if it is production and consumption tps monitoring, brokerAddr is also required.

```java
// 参数为监控器名、伸缩组名、伸缩组、定时执行的cron表达式、统计持续时间（单位：秒）、统计方式（最大、最小、平均值）、扩容阈值、缩容阈值、连续达到阈值多少次触发伸缩
CpuMonitorRunnable cpuMonitorRunnable = new CpuMonitorRunnable("CPU-CONSUMER-MONITER", "CONSUMER-SCALE", consumerScalingGroup, "0/5 * * * * ?", 3, MonitorMethod.AVG, 80, 20, 3);

// 如果是生产和消费tps监控，还需提供brokerAddr
ProduceTpsMonitor produceTpsMonitor = new ProduceTpsMonitor("PRODUCE_TPS_MONITOR", "PRODUCER_SCALING_GROUP", producerScalingGroup, "0/5 * * * * ?", 3, MonitorMethod.AVG, 0, -1, 3, "192.168.20.100:10911");
```

4. Create a MonitorManager, and call its startMonitor method to start the monitor created in step 3, and auto scaling starts.

```java
MonitorManager monitorManager = new MonitorManager();
monitorManager.startMonitor(cpuMonitorRunnable);
```

5. The entire code for enabling consumer elastic scaling is as follows. You can also view the ConsumerScalingTest class in the scaling package under the test package.

```java
    @Test
    public void testScalingConsumer() {
        Map<String, String> topicAndTag = new HashMap<>();
        topicAndTag.put("SCALE", "*");
        // 创建消费者启动模板
        ConsumerStartingTemplate consumerStartingTemplate = new ConsumerStartingTemplate("SCALE_TEST_CONSUMER_GROUP", "192.168.20.100:9876", ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET,
                topicAndTag, MessageModel.CLUSTERING, new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                // 逐条消费消息
                for (MessageExt msg : msgs) {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                // 返回消费状态:消费成功
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        // 创建消费者伸缩组
        ConsumerScalingGroup consumerScalingGroup = new ConsumerScalingGroup("CONSUMER-SCALE", 0, 3, 5,
                                                        consumerStartingTemplate, new ConsumerRemoveStrategy().getOldestRemoveStrategy());
        // 创建资源监控器
        CpuMonitorRunnable cpuMonitorRunnable = new CpuMonitorRunnable("CPU-CONSUMER-MONITER", "CONSUMER-SCALE", consumerScalingGroup,
                                                    "0/5 * * * * ?", 3, MonitorMethod.AVG, 80, 20, 3);
        // 这里只是为了在windows系统下进行测试，假设cpu利用率一直为90%
        cpuMonitorRunnable.setOs("windows");
        cpuMonitorRunnable.setCmd("echo 90");
//        cpuMonitorRunnable.setCmd("echo 10");
        // 创建MonitorManager，并启动monitor
        MonitorManager monitorManager = new MonitorManager();
        monitorManager.startMonitor(cpuMonitorRunnable);

        // while true是避免程序结束，为了打印出日志信息
        while (true) {

        }
    }
```

#### Producer Elastic Scaling

1. To create a producer startup template, the user needs to provide the producer group name, NameServer address, number of retries after sending failures, and how many queues a topic corresponds to.

```java
// 参数为生产者组名、NameServer地址、发送失败后的重试次数、一个topic对应多少个Queue
ProducerStartingTemplate producerStartingTemplate = new ProducerStartingTemplate("SCALE_PRODUCER_GROUP", "192.168.20.100:9876", 0, 2);
```

2. To create a producer scaling group, the user needs to provide the scaling group name, minimum number of producer instances, maximum number of producer instances, cooling time, producer startup template, and shrinking strategy (remove the latest or earliest created producer, ProducerRemoveStrategy provides the getOldestRemoveStrategy() and getNewestRemoveStrategy() methods).

```java
// 参数为伸缩组名、最小生产者实例数、最大生产者实例数、冷却时间、生产者启动模板、收缩策略
ProducerScalingGroup producerScalingGroup = new ProducerScalingGroup("PRODUCER_SCALING_GROUP", 0, 3, 5, producerStartingTemplate, new ProducerRemoveStrategy().getOldestRemoveStrategy());
```

3. Create a monitor class. Here we take the production tps monitor as an example. The user needs to provide the monitor name, scaling group name, scaling group, cron expression for timing execution, statistical duration (unit: second), and statistical method (maximum , minimum, average value), expansion threshold, shrinking threshold, how many times the threshold is reached consecutively to trigger scaling, and brokerAddr.

```java
// 参数为监控器名、伸缩组名、伸缩组、定时执行的cron表达式、统计持续时间（单位：秒）、统计方式（最大、最小、平均值）、扩容阈值、缩容阈值、连续达到阈值多少次触发伸缩，brokerAddr
ProduceTpsMonitor produceTpsMonitor = new ProduceTpsMonitor("PRODUCE_TPS_MONITOR", "PRODUCER_SCALING_GROUP", producerScalingGroup, "0/5 * * * * ?", 3, MonitorMethod.AVG, 0, -1, 3, "192.168.20.100:10911");
```

4. Create a MonitorManager and start the Monitor, and auto scaling starts.

```java
MonitorManager monitorManager = new MonitorManager();
monitorManager.startMonitor(produceTpsMonitor);
```

5. The entire code for enabling producer auto scaling is as follows. You can also view the ProducerScalingTest class in the scaling package under the test package.

```java
@Test
public void testScalingProducer() {
    // 创建生产者启动模板
    ProducerStartingTemplate producerStartingTemplate = new ProducerStartingTemplate("SCALE_PRODUCER_GROUP",
            "192.168.20.100:9876", 0, 2);
    // 创建生产者伸缩组
    ProducerScalingGroup producerScalingGroup = new ProducerScalingGroup("PRODUCER_SCALING_GROUP", 0, 3, 5, producerStartingTemplate, new ProducerRemoveStrategy().getOldestRemoveStrategy());
    // 创建资源监控器
    ProduceTpsMonitor produceTpsMonitor = new ProduceTpsMonitor("PRODUCE_TPS_MONITOR", "PRODUCER_SCALING_GROUP", producerScalingGroup,
            "0/5 * * * * ?", 3, MonitorMethod.AVG, 0, -1, 3, "192.168.20.100:10911");
    // 创建MonitorManager并启动Monitor
    MonitorManager monitorManager = new MonitorManager();
    monitorManager.startMonitor(produceTpsMonitor);

    // while true是避免程序结束，为了打印出日志信息
    while (true) {

    }
}
```



## Other functions

Other functions are implemented in the service package, and the entry is the impl package, which provides BrokerServiceImpl, MessageServiceImpl, QueueServiceImpl, and TopicServiceImpl. The following three functions are listed as examples.



### Get the logical position of the message in the queue

```java
// 创建MessageServiceImpl
MessageServiceImpl messageService = new MessageServiceImpl();
// 在生产者发送成功消息后，将SendResult作为参数，调用getMessageLogicPositionInQueueAfterSend方法可以得到消息在队列中的逻辑位置
Long logicPos = messageService.getMessageLogicPositionInQueueAfterSend(sendResult);
```



### Get production and consumption tps

```java
// 创建BrokerServiceImpl
BrokerServiceImpl brokerServiceImpl = new BrokerServiceImpl();
// 调用getProduceAndConsumeTpsByBrokerAddr(String brokerAddr)方法获取生产和消费的tps,返回值为一个Map,对应key为"produceTps"的value为生产tps，对应key为"consumeTps"的value为消费tps
Map<String, Long> map = brokerServiceImpl.getProduceAndConsumeTpsByBrokerAddr("192.168.20.100:10911");
Long produceTps = map.get("produceTps");
Long consumeTps = map.get("consumeTps");
```



### Get the corresponding relationship between the queue and its consumers

```java
// 创建QueueServiceImpl
QueueServiceImpl queueServiceImpl = new QueueServiceImpl();
// 调用getQueueConsumerRelationByConsumerGroup(String consumerGroupName)方法通过消费者组名获取队列及其所属的消费者的对应关系，返回值为Map<MessageQueue, String>
queueServiceImpl.getQueueConsumerRelationByConsumerGroup(xxx);
```



## TODO List

1. For idempotent deduplication, add a lightweight solution. For a system, it has tolerance. For example, repeated messages will only appear within 5 minutes, so you can directly use redis to prevent repeated consumption. Redis stores message information, and the survival time is set to 6 minutes. As long as the message can be found in redis, the current message is a repeated message, otherwise it is not.

2. Design elastic scaling to scale producers and consumers in producer and consumer clusters through RPC communication or network communication.

3. Several commonly used functions are implemented through the RocketMQ client in the rocketmq-tools package. Calling the API is actually performing network communication, that is, the obtained information is not real-time information, but the information of the previous snapshot status. Fetching real-time status will be considered later.