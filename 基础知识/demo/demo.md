一、消息传递语义：
三种，至少一次，至多一次，精确一次

1、at lest once：消息不丢，但可能重复

2、at most once：消息会丢，但不会重复

3、Exactly Once：消息不丢，也不重复。

保证消息不丢、消息不重复

消息不丢：副本机制+ack，可以保证消息不丢。

数据重复：brocker保存了消息之后，在发送ack之前宕机了，producer认为消息没有发送成功进行重试，导致数据重复。

数据乱序：前一条消息发送失败，后一条消息发送成功，前一条又重试，成功了，导致数据乱序。

二、消息一致性保证方案：
主要就是保证Exactly Once，即：数据不丢、数据不重复

0.11之前的kafka版本：保证消息不丢，要在消息发送端和消费端都要进行保证。保证消息不重复，就是要对消息幂等，即去重

1、消息发送端producer保证Exactly Once语义：

request.required.acks 设置数据可靠性级别：

request.required.acks=1：当且仅当leader收到消息后返回commit确认信号后，消息发送成功，也就是说只需要leader确认。但有弊端，leader宕机，也就是还没有将消息同步到follower，这是会发生消息丢失。

request.required.acks=0：消息发送了，即认为成功，可靠性最低。

request.required.acks=-1：发送端等待isr列表所有的成员确认消息，才算成功，可靠性最高延迟最大。

2、消息消费端consumer保证Exactly Once语义：

（1）消息不丢

       消费者关闭自动提交，enable.auto.commit:false，消费者收到消息处理完业务逻辑后，再手动提交commitSync offersets。这样可以保证消费者即使在消息处理过程中挂掉，下次重启，也可以从之前的offersets进行消费，可以保证消息不丢。

更新offset操作和批处理操作放在同一个redis事物中，保证消息不重复处理，使用事务会降低消息处理的效率，若数据量小，可以使用事务，数据量大就不使用了。

（2）消息幂等：主要借助于业务系统本身的业务处理或大数据组件幂等。如：hbase 、elasticsearch幂等。

幂等数据库：将消息从kafka消费出来，保存到hbase中，使用id主键+时间戳，只有插入成功后才往 kafka 中持久化 offset。这样的好处是，如果在中间任意一个阶段发生报错，程序恢复后都会从上一次持久化 offset 的位置开始消费数据，而不会造成数据丢失。如果中途有重复消费的数据，则插入 hbase 的 rowkey 是相同的，数据只会覆盖不会重复，最终达到数据一致。

使用redis去重：

数据量小，可以使用redis的set进行去重，并存储相关数据，如：日活用户
数据量大，可以使用redis的bitmap进行去重，并存储相关数据，日：日活用户
在之前的旧版本中，Kafka只能支持两种语义：At most once和At least once。At most once保证消息不会重复，但是可能会丢失。在实践中，很有有业务会选择这种方式。At least once保证消息不会丢失，但是可能会重复，业务在处理消息需要进行去重。

Kafka在0.11.0.0版本支持增加了对幂等的支持。幂等是针对生产者角度的特性。幂等可以保证上生产者发送的消息，不会丢失，而且不会重复。

Kafka幂等性介绍与源码实现 - 简书

Kafka的消息会丢失和重复吗？——如何实现Kafka精确传递一次语义

Kafka的消息会丢失和重复吗？——如何实现Kafka精确传递一次语义 - 云+社区 - 腾讯云

3、kafka手动维护偏移量

在项目中，kafka和sparkStream采用的是直连方式，使用的是kafka基础的api，因此需要手动维护偏移量。将偏移量保存在mysql中。

程序运行时，先去mysql中查询偏移量，判断是否是程序第一次启动，若是第一次启动，就是不指定偏移量，重头读取kafka数据。若是非第一次启动，即从mysql中有偏移量。此时还要对比数据库中的偏移量和kafka现在每个分区的最早偏移量getEarliestLeaderOffsets，因为kafka数据默认是保存七天，也就是偏移量有效期就是七天。若数据库中的偏移量没过期，那就从数据库保存的偏移量开始读。若过期了，那就从现在最新的开始读。

       这里出现一个问题，kafka的patition分区数不一定不变，有时候就是为了提升spark Streaming的并行处理的能力，这时要必须增加kafka的patition分区数以对应spark Streaming的executor数，--num- executor这个主要设置即可，因为patition分区数要等于executor的数量，大了小了都不好。而新增分区的偏移量若没有及时保存在数据库上的话，就会出现数据丢失，消费不到新增分区的数据。

     这里的解决方式，就是每次启动流程序前，对比一下当前我们自己保存的kafka的分区的个数和从zookeeper里面的存的topic的patition分区个数是否一致，如果不一致，就把新增的分区给添加到我们自己保存的信息中，并发偏移量初始化成 0，这样以来在程序启动后，就会自动识别新增分区的数据。 

参考博客：Kafka偏移量维护中的坑  自己维护kafka_offset中的坑 - 简书

三、kafka消息丢失场景
1、Kafka消息丢失的情况：

（1）auto.commit.enable=true，消费端自动提交offersets设置为true，当消费者拉到消息之后，还没有处理完 commit interval 提交间隔就到了，提交了offersets。这时consummer又挂了，重启后，从下一个offersets开始消费，之前的消息丢失了。

（2）网络负载高、磁盘很忙，写入失败，又没有设置消息重试，导致数据丢失。

（3）磁盘坏了已落盘数据丢失。

（4）单 批 数 据 的 长 度 超 过 限 制 会 丢 失 数 据 ， 报kafka.common.Mess3.ageSizeTooLargeException异常

2、Kafka避免消息丢失的解决方案：

（1）设置auto.commit.enable=false，每次处理完手动提交。确保消息真的被消费并处理完成。

（2）kafka 一定要配置上消息重试的机制，并且重试的时间间隔一定要长一些，默认 1 秒钟不符合生产环境（网络中断时间有可能超过 1秒）。

（3）配置多个副本，保证数据的完整性。

（4）合理设置flush间隔。kafka 的数据一开始就是存储在 PageCache 上的，定期 flush 到磁盘上的，也就是说，不是每个消息都被存储在磁盘了，如果出现断电或者机器故障等，PageCache 上的数据就丢。可以通过 log.flush.interval.messages 和 log.flush.interval.ms 来配置 flush 间隔，interval大丢的数据多些，小会影响性能但在 0.本，可以通过 replica机制保证数据不丢，代价就是需要更多资源，尤其是磁盘资源，kafka 当前支持 GZip 和 Snappy压缩，来缓解这个问题 是否使用 replica 取决于在可靠性和资源代价之间的 balance。

http://bigdata-star.com/archives/1507

四、kafka消息重复场景
其实kafka的重复消费问题究其底层根本原因就是：已经消费了数据，但是offset没提交(kafka没有或者不知道该数据已经被消费)。 基于这种原因总结以下几个易造成重复消费的配置：
原因1：强行kill线程，导致消费后的数据，offset没有提交（消费系统宕机、重启等）。
原因2：设置offset为自动提交，关闭kafka时，如果在close之前，调用 consumer.unsubscribe() 则有可能部分offset没提交，下次重启会重复消费。例如：



try {
consumer.unsubscribe();
} catch (Exception e) {
}

try {
consumer.close();
} catch (Exception e) {
}


上面代码会导致部分offset没提交，下次启动时会重复消费。
原因3:（重复消费最常见的原因）：消费后的数据，当offset还没有提交时，partition就断开连接。比如，通常会遇到消费的数据，处理很耗时，导致超过了Kafka的session timeout时间（0.10.x版本默认是30秒），那么就会re-blance重平衡，此时有一定几率offset没提交，会导致重平衡后重复消费。
原因4：当消费者重新分配partition的时候，可能出现从头开始消费的情况，导致重发问题。
原因5：当消费者消费的速度很慢的时候，可能在一个session周期内还未完成，导致心跳机制检测报告出问题。

问题描述：

       我们系统压测过程中出现下面问题：异常rebalance，而且平均间隔3到5分钟就会触发rebalance，分析日志发现比较严重。错误日志如下：




08-09 11:01:11 131 pool-7-thread-3 ERROR [] -

commit failed

org.apache.kafka.clients.consumer.CommitFailedException: Commit cannot be completed since the group has already rebalanced and assigned the partitions to another member. This means that the time between subsequent calls to poll() was longer than the configured max.poll.interval.ms, which typically implies that the poll loop is spending too much time message processing. You can address this either by increasing the session timeout or by reducing the maximum size of batches returned in poll() with max.poll.records.

at org.apache.kafka.clients.consumer.internals.ConsumerCoordinator.sendOffsetCommitRequest(ConsumerCoordinator.java:713) ~[MsgAgent-jar-with-dependencies.jar:na]

at org.apache.kafka.clients.consumer.internals.ConsumerCoordinator.commitOffsetsSync(ConsumerCoordinator.java:596) ~[MsgAgent-jar-with-dependencies.jar:na]

at org.apache.kafka.clients.consumer.KafkaConsumer.commitSync(KafkaConsumer.java:1218) ~[MsgAgent-jar-with-dependencies.jar:na]

at com.today.eventbus.common.MsgConsumer.run(MsgConsumer.java:121) ~[MsgAgent-jar-with-dependencies.jar:na]

at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [na:1.8.0_161]

at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [na:1.8.0_161]

at java.lang.Thread.run(Thread.java:748) [na:1.8.0_161]



       这个错误的意思是，消费者在处理完一批poll的消息后，在同步提交偏移量给broker时报的错。初步分析日志是由于当前消费者线程消费的分区已经被broker给回收了，因为kafka认为这个消费者死了，那么为什么呢？

问题分析：

这里就涉及到问题是消费者在创建时会有一个属性max.poll.interval.ms，
该属性意思为kafka消费者在每一轮poll()调用之间的最大延迟,消费者在获取更多记录之前可以空闲的时间量的上限。如果此超时时间期满之前poll()没有被再次调用，则消费者被视为失败，并且分组将重新平衡，以便将分区重新分配给别的成员。

       如上图，在while循环里，我们会循环调用poll拉取broker中的最新消息。每次拉取后，会有一段处理时长，处理完成后，会进行下一轮poll。引入该配置的用途是，限制两次poll之间的间隔，消息处理逻辑太重，每一条消息处理时间较长，但是在这次poll()到下一轮poll()时间不能超过该配置间隔，协调器会明确地让使用者离开组，并触发新一轮的再平衡。
max.poll.interval.ms默认间隔时间为300s。

测试对应表格：



kafka的生产消费具体参数配置实例如下：



kafka.consumer.zookeeper.connect=zookeeper-ip:2181

kafka.consumer.servers=kafka-ip:9092

kafka.consumer.enable.auto.commit=true

kafka.consumer.session.timeout=6000

kafka.consumer.auto.commit.interval=100

kafka.consumer.auto.offset.reset=latest

kafka.consumer.topic=test

kafka.consumer.group.id=test

kafka.consumer.concurrency=10


kafka.producer.servers=kafka-ip:9092

kafka.producer.retries=0

kafka.producer.batch.size=4096

kafka.producer.linger=1
 