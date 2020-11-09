## 生产故障现象
- 1、 kafka消费在微服务启动后，正常消费，经过一段时间后，线程死掉，无法消费
- 2、 在kafka监控管理后台看到，进入当前groupId=mps_ticket_data_send_sync_track_status 查看当前topic=mps_ticket_data_send_sync_track_status,发现活动消费者线程都为0；
查看Consumers Offsets 中Lag项，不为0，且累计增加；
- 3、 进入微服务查看日志，在某个时间点后无任何消费者相关信息。
## 分析思路
- 1、进入服务器，jps查进程号，top -Hp xx 查看进程相关信息：cpu以及内存线程等，一切正常；
- 2、jstack导出当前进行运行信息，并用消费者线程名称模糊匹配，无任何异常信息。
- 3、kafka服务端其他topic运行正常；
- 4、本地模拟消费该有问题topic可正常消费；
- 5、联系运维，先对微服务进行灰度重启（共3台，先启动一台），服务重启后消费者可以正常消费消息。但运行一段时间后（10分钟左右），线程卡死，再无任何消费行为。
- 6、查看在卡死之后的业务日志，发现如下异常信息：
```
2020-11-04 20:14:31.431 [couponSyn-30949427610350292] ERROR cn.wonhigh.o2o.coupon.kafka.retailkafka.CouponConsumer:148 - Commit cannot be completed since the group has already rebalanced and assigned the partitions to another member. This means that the time between subsequent calls to poll() was longer than the configured session.timeout.ms, which typically implies that the poll loop is spending too much time message processing. You can address this either by increasing the session timeout or by reducing the maximum size of batches returned in poll() with max.poll.records.
org.apache.kafka.clients.consumer.CommitFailedException: Commit cannot be completed since the group has already rebalanced and assigned the partitions to another member. This means that the time between subsequent calls to poll() was longer than the configured session.timeout.ms, which typically implies that the poll loop is spending too much time message processing. You can address this either by increasing the session timeout or by reducing the maximum size of batches returned in poll() with max.poll.records.
at org.apache.kafka.clients.consumer.internals.ConsumerCoordinator$OffsetCommitResponseHandler.handle(ConsumerCoordinator.java:578) ~[kafka-clients-0.10.0.0.jar:na]
at org.apache.kafka.clients.consumer.internals.ConsumerCoordinator$OffsetCommitResponseHandler.handle(ConsumerCoordinator.java:519) ~[kafka-clients-0.10.0.0.jar:na]
at org.apache.kafka.clients.consumer.internals.AbstractCoordinator$CoordinatorResponseHandler.onSuccess(AbstractCoordinator.java:679) ~[kafka-clients-0.10.0.0.jar:na]
at org.apache.kafka.clients.consumer.internals.AbstractCoordinator$CoordinatorResponseHandler.onSuccess(AbstractCoordinator.java:658) ~[kafka-clients-0.10.0.0.jar:na]
at org.apache.kafka.clients.consumer.internals.RequestFuture$1.onSuccess(RequestFuture.java:167) ~[kafka-clients-0.10.0.0.jar:na]
at org.apache.kafka.clients.consumer.internals.RequestFuture.fireSuccess(RequestFuture.java:133) ~[kafka-clients-0.10.0.0.jar:na]
at org.apache.kafka.clients.consumer.internals.RequestFuture.complete(RequestFuture.java:107) ~[kafka-clients-0.10.0.0.jar:na]
at org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient$RequestFutureCompletionHandler.onComplete(ConsumerNetworkClient.java:426) ~[kafka-clients-0.10.0.0.jar:na]
at org.apache.kafka.clients.NetworkClient.poll(NetworkClient.java:278) ~[kafka-clients-0.10.0.0.jar:na]
at org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient.clientPoll(ConsumerNetworkClient.java:360) ~[kafka-clients-0.10.0.0.jar:na]
at org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient.poll(ConsumerNetworkClient.java:224) ~[kafka-clients-0.10.0.0.jar:na]
at org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient.poll(ConsumerNetworkClient.java:192) ~[kafka-clients-0.10.0.0.jar:na]
at org.apache.kafka.clients.consumer.internals.ConsumerNetworkClient.poll(ConsumerNetworkClient.java:163) ~[kafka-clients-0.10.0.0.jar:na]
at org.apache.kafka.clients.consumer.internals.ConsumerCoordinator.commitOffsetsSync(ConsumerCoordinator.java:404) ~[kafka-clients-0.10.0.0.jar:na]
at org.apache.kafka.clients.consumer.KafkaConsumer.commitSync(KafkaConsumer.java:1058) ~[kafka-clients-0.10.0.0.jar:na]
at org.apache.kafka.clients.consumer.KafkaConsumer.commitSync(KafkaConsumer.java:1026) ~[kafka-clients-0.10.0.0.jar:na]
at cn.wonhigh.o2o.coupon.kafka.retailkafka.CouponConsumer$PullConsumerData.run(CouponConsumer.java:144) ~[member-outside-process-1.0-SNAPSHOT.jar:1.0-SNAPSHOT]
at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [na:1.8.0_192]
at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [na:1.8.0_192]
at java.lang.Thread.run(Thread.java:748) [na:1.8.0_192]
```
- 7、根据错误信息，可以得知因为使用分组消费，消费组内进行了重平衡，导致commit失败。

## 解决方案
- 1、重平衡解决方案：poll的间隔时间超过了会话时间，此时将会话时间调长；也有可能是poll拉取的消息过多，导致业务处理时间过长，超过了会话时间，此时可以通过减少最大拉取消息数量参数，减少当前批业务处理时间。
```
当前事故系统没有设置拉取最大条数；但从日志中分析出，一次循环拉取数量超过500条有1800多条；
# 默认拉500条数据
# Consumer每次调用poll()时取到的records的最大数。调整为5，加快业务处理速度
max.poll.records = 5

```

## 结果
- 1、重设参数后发版，程序刚开是运行正常，之后遇到的情况和修复前一样卡死，kafka服务端正常。
- 2、重新检查业务日志没有出现重平衡异常信息，但也没有发现其他异常信息或错误信息；最后在stdout.log中发现如下错误信息：
```
NoSuchMethodError:没有找到此方法错误
ReUtil.findAll()
<!--关注运行时抛出的 NoSuchMethodError 错误，该错误轻则导致程序异常终止，严重时甚至会产生不可预知的程序结果。
-->
<!--参考：https://blog.csdn.net/wypblog/article/details/102951861-->
```
- 3、根据此错误，找到此方法所在的类，查看当前项目引用版本如下：
```
        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
            <version>4.1.19</version>
        </dependency>
```
- 4、分析此包hutool-all发现存在版本冲突问题，系统中还存在另一个版本5.1.0（此包通过一个基础包引入）。确定是版本冲突导致抛出此错误，此错误导致整个kafka消费者线程死掉。

## 调整解决方案
- 1、hutool-all.jar注释掉4.1.19版本，保存5.1.0版本，再次发版。消费者正常消费，再无卡死状况。
