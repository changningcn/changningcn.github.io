---
layout: post
title: Logstash
tag: ElasticSearch
---
## Logstash
[logstash 目录结构](https://www.elastic.co/guide/en/logstash/current/config-setting-files.html)

## Logstash input plugin - kafka
[kafka input plugin reference](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-kafka.html)

从 kafka topic 中读取 events，logstash 作为消费者进行 group 的管理，并使用 kafka 默认的 offset 管理策略。

Logstash 实例默认使用一个逻辑组(`group_id => "logstash"`)来订阅 Kafka topic，使用者可以运行多个线程来提高读取吞吐量。或者，直接使用相同的`group_id`运行多个 Logstash 实例，以跨物理机分散负载。主题中的消息将分发到具有相同`group_id`的所有 Logstash 实例。

理想情况下，线程数应该与分区数量一样多以实现完美平衡 - 线程多于分区意味着某些线程将处于空闲状态

kafka 的 metadata：

* \[@metadata]\[kafka]\[topic]: Original Kafka topic from where the message was consumed.
* \[@metadata]\[kafka]\[consumer_group]: Consumer group
* \[@metadata]\[kafka]\[partition]: Partition info for this message.
* \[@metadata]\[kafka]\[offset]: Original record offset for this message.
* \[@metadata]\[kafka]\[key]: Record key, if any.
* \[@metadata]\[kafka]\[timestamp]: Timestamp when this message was received by the Kafka broker.

配置 kafka input:

```shell
input {
  kafka {
    id => "kafka_inuput_logstash_person"
    bootstrap_servers => "kafka:9999"
    client_id => "test"
    group_id => "test"
    topics => ["logstash_person"]
    auto_offset_reset => "latest"
    consumer_threads => 5
    decorate_events => true
    type => "person"
  }
  kafka {
    id => "kafka_inuput_logstash_cat"
    bootstrap_servers => "kafka:9999"
    client_id => "test"
    group_id => "test"
    topics => ["logstash_cat"]
    auto_offset_reset => "latest"
    consumer_threads => 5
    decorate_events => true
    type => "cat"
  }
}
filter {
  if [type] == "person" {
    grok {
      match => { "message" => "%{WORD:name} %{DATE:birthday} %{IP:ip}" }
      remove_field => "message"
    }
  }
  else if [type] == "cat" {
    grok {
      match => { "message" => "%{WORD:kind} %{WORD:master} %{NUMBER:age}" }
    }
    mutate {
      add_field => { "read_timestamp" => "%{@timestamp}" }
    }
  }
}
output {
  elasticsearch {
    hosts => ["http://es_node:9200"]
    index => "logstash-%{[type]}-%{+YYYY.MM.dd}"
    manage_template => false
  }
} 
```

* id: input plugin 的唯一标识符
* bootstrap_server: kafka 节点地址
* client_id: 发出请求时传递给服务器的id字符串，这样包含逻辑的应用程序可以跟踪请求的来源。
* group_id: 消费者 groupID
* topics: kafka topic，数组类型
* auto_offset_reset: 当 Kafka 中没有初始偏移量或偏移量超出范围时的策略。`earliest`: 从头开始消费；`latest`: 从最新的offset开始消费；`none`: 如果没有找到消费者组的先前偏移量，则向消费者抛出异常；`anything else`: 直接向消费者抛出异常。
* consumer_threads: 消费者线程数
* decorate_events: 此属性会将当前 topic、offset、group、partition 等信息也带到 message 中


## Logstash filter plugin - grok
grok 是一个可以将非结构化的日志解析成为结构化和可查询数据的 filter plugin。[grok filter plugin reference](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html)

语法`%{SYNTAX:SEMANTIC}`，其中`SYNTAX`是匹配内容的格式，`SEMANTIC`是唯一标识符，可以理解为`key`。

例如：

```shell
# 日志文件 http.log 格式如下
55.3.244.1 GET /index.html 15824 0.043

filter {
  grok {
      match => { "message" => "%{IP:client} %{WORD:method} %{URIPATHPARAM:request} %{NUMBER:bytes} %{NUMBER:duration}" }
  }
}

# 解析后，event 将会添加一些额外的字段：
client: 55.3.244.1
method: GET
request: /index.html
bytes: 15824
duration: 0.043
```
grok 已经支持的 pattern 可以参考[https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns](https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns)

grok 是基于正则表达式的，因此任何正则表达式在 grok 当中也是有效的。使用的正则表达式库是 Oniguruma。可以去 [Oniguruma 官网](https://github.com/kkos/oniguruma/blob/master/doc/RE)查看所支持的正则表达式语法。

有时候 logstash 没有满足要求的 pattern，可以自定自己的正则表达式，然后为匹配项定义一个字段。

语法：`(?<field_name> pattern)`，注意不需要再加`%{}`了

例如：
```shell
filter {
  grok {
    patterns_dir => ["./patterns"]
    match => { "message" => "%{SYSLOGBASE} (?<id>[0-9A-F]{10,11}): %{GREEDYDATA:syslog_message}" }
  }
}
```

除此之外还可以定义一个 pattern 文件。
```shell
vim ./patterns/my_pattern
ID [0-9A-F]{10,11}

# 然后通过 patterns_dir 说明自定义的 pattern 文件所在文件夹路径
filter {
  grok {
    patterns_dir => ["./patterns"]
    match => { "message" => "%{SYSLOGBASE} %{ID:id}: %{GREEDYDATA:syslog_message}" }
  }
}
```

[测试 grok 语法正确性](http://grokdebug.herokuapp.com/)

grok pattern 可以将 string 转换成数字类型，但是目前只支持 int 和 float，语法：`%{NUMBER:num:int}`

## Logstash filter plugin - mutate
mutate 用于重命名，删除，替换和修改事件中的字段。[mutate filter plugin reference](https://www.elastic.co/guide/en/logstash/current/plugins-filters-mutate.html)

## 在 Logstash 的 config 文件中访问 metadata
```shell
input {
  beats {
    port => 5044
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}" 
  }
}
```

## 在 Logstash 的 config 文件中访问 fields

```shell
input {
    beats {
        port => "5044"
    }
}

output {
    stdout { codec => rubydebug }
    kafka {
        topic_id => "%{[fields][topic_id]}"
        codec => plain {
            format => "%{message}"
            charset => "UTF-8"
        }
        bootstrap_server    s => "kafka:9999"
    }
}
```

## 启动

```shell
# 测试配置文件
bin/logstash -f pipeline1.conf --config.test_and_exit

# 启动 logstash 
# --config.reload.automatic 不需要重启就可以加载被修改的配置文件
nohup bin/logstash -f pipeline1.conf --config.reload.automatic 2>&1 &

# 查看 logstash 运行日志，日志路径和格式可以通过 config/log4j2.properties 控制
tail -100f logs/logstash-plain.log

# 注意如果想同时启动多个 logstash 实例，需要修改 config/logstash.yml 中的 path.data
```

## 异常
### OutOfMemoryError
```console
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid26719.hprof ...
Heap dump file created [1385285026 bytes in 5.825 secs]
Exception in thread "defaultEventExecutorGroup-4-1" java.lang.OutOfMemoryError: Java heap space
Exception: java.lang.OutOfMemoryError thrown from the UncaughtExceptionHandler in thread "kafka-producer-network-thread | producer-1"

Exception in thread "defaultEventExecutorGroup-4-2" java.lang.OutOfMemoryError: Java heap space
Exception in thread "LogStash::Runner" java.lang.OutOfMemoryError: Java heap space
Exception in thread "Ruby-0-Thread-1: /root/elk_changning/logstash-6.2.3-ship/lib/bootstrap/environment.rb:6" java.lang.OutOfMemoryError: Java heap space
Exception in thread "nioEventLoopGroup-3-3" java.lang.OutOfMemoryError: Java heap space
```
通过 JVM 参数`--XX:-HeapDumpOnOutOfMemoryError`可以让 JVM 在出现内存溢出时 dump 出当前的内存转储快照，如上面的异常(第二行)所示。

通过JProfiler等工具分析内存溢出的原因