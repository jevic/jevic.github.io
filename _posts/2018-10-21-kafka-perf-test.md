---
layout: post
categories: Kafka Bigdata
title: kafka 压力测试
date: 2018-10-21 18:10:46 +0800
description: kafka
keywords: kafka
---


kafka 内置提供了压测脚本

## kafka 版本
```
$ find /opt/kafka/libs/ -name \*kafka_\* | head -1 | grep -o '\kafka[^\n]*'
kafka/libs/kafka_2.11-1.0.0.2.6.5.1050-37-javadoc.jar
```
---

## Producer测试
```
$ ./bin/kafka-producer-perf-test.sh \
--topic test \
--num-records 1000000 \
--record-size 3000 \
--throughput 20000 \
--producer-props bootstrap.servers=192.168.25.198:6667,192.168.25.199:6667,192.168.25.200:6667,192.168.25.201:6667,192.168.25.202:6667

99914 records sent, 19978.8 records/sec (57.16 MB/sec), 124.9 ms avg latency, 566.0 max latency.
99292 records sent, 19858.4 records/sec (56.82 MB/sec), 7.1 ms avg latency, 214.0 max latency.
96498 records sent, 19299.6 records/sec (55.22 MB/sec), 117.3 ms avg latency, 1096.0 max latency.
95601 records sent, 19120.2 records/sec (54.70 MB/sec), 269.1 ms avg latency, 2194.0 max latency.
108777 records sent, 21755.4 records/sec (62.24 MB/sec), 340.9 ms avg latency, 2665.0 max latency.
100020 records sent, 20004.0 records/sec (57.23 MB/sec), 1.3 ms avg latency, 20.0 max latency.
100051 records sent, 20010.2 records/sec (57.25 MB/sec), 1.3 ms avg latency, 20.0 max latency.
99987 records sent, 19997.4 records/sec (57.21 MB/sec), 1.3 ms avg latency, 27.0 max latency.
100036 records sent, 20007.2 records/sec (57.24 MB/sec), 1.3 ms avg latency, 21.0 max latency.
1000000 records sent, 19992.402887 records/sec (57.20 MB/sec), 87.97 ms avg latency, 2665.00 ms max latency, 1 ms 50th, 500 ms 95th, 2202 ms 99th, 2613 ms 99.9th.
```

--topic topic名称，本例为test_perf

--num-records 总共需要发送的消息数，本例为1000000

--record-size 每个记录的字节数，本例为1000

--throughput 每秒钟发送的记录数，本例为20000

--producer-props bootstrap.servers=localhost:9092 发送端的配置信息


>上面示例可以看出 平均每秒写入57.20MB数据,大概是19992条消息

>每次写入的平均延迟是: 87.98 毫秒

>最大延迟是: 2665.00 毫秒

---

## Consumer 测试

```
./bin/kafka-consumer-perf-test.sh 
--topic test \
--fetch-size 1048576 \
--messages 1000000 \
--threads 20 \
--hide-header \
--broker-list 192.168.25.198:6667,192.168.25.199:6667,192.168.25.200:6667,192.168.25.201:6667,192.168.25.202:6667
```

--topic 指定topic的名称，本例为test_perf

--fetch-size 指定每次fetch的数据的大小，本例为1048576，也就是1M

--messages 总共要消费的消息个数，本例为1000000，100w

- 删除topic

```
./kafka-topics.sh --zookeeper 192.168.25.198:2181 --delete --topic test
```


---

- 可逐一增大数量级以及消息个数等参数 对比结果;
- 详细的参数信息直接执行脚本查看帮助信息;
