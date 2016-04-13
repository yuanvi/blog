---
published: true
title: storm源码分析 - ShellBolt实现
layout: post
tags: [storm, 源码分析, java]
categories: [java, 分布式计算]
---

使用storm的过程中，可能在业务的某个处理节点上，只是想把原来实现的代码简单修改就快速迁移到blot或者spout中，storm也提供了对这种需求的支持。在源码的storm-multilang目录下，提供了javascript、python和ruby的多语言库，迁移业务时，只需要用这些库对输入输出进行封装即可。实际上，storm的通过子进程的标准输入输出来进行通信，也就是说，只要我们封装的子进程满足这个通信协议，就可以使用ShellBolt来运行。先简单说明一个使用ShellBlot的例子：

```java
// src/main/java/bolts/WordBolt.java
package bolts;

import java.util.Map;
import backtype.storm.topology.OutputFieldsDeclarer;
import backtype.storm.topology.IRichBolt;
import backtype.storm.tuple.Fields;
import backtype.storm.task.ShellBolt;

public class WordBolt extends ShellBolt implements IRichBolt {

    public WordBolt(){
        // wordbolt.py在pom.xml的resources指定的目录中
        super("python", "wordblot.py");
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("time"));
    }

    @Override
    public Map<String, Object> getComponentConfiguration() {
        return null;
    }
}
```

在pom.xml文件中指定resources文件的位置，并且在其中实现wordblot.py，topology运行时就会创建python wordblot.py的子进程，并将接收的流转换为json数据通过标准输入传递给它，并且collect标准输出的数据并处理。这里分析storm-multilang/python/resources/storm.py是如何封装子进程的输入输出的。

storm.py中实现了3个Class，Bolt、BasicBolt和Spout，其中Bolt和BasicBolt的区别在于BasicBolt会自动ack收到的Tuple，在使用时继承并重写相应的方法即可，storm.py会通过全局变量MODE来自动区分是Bolt还是Spout。

上面说到，ShellBolt和子进程的通信格式是json，其中子进程发送json中，必须包含command字段来告诉ShellBolt这是一个什么类型的消息，command字段可能的值有：

* ack: ack Tuple
* fail: fail Tuple
* log: 打印一条log
* error: 同log，log级别是error，打印的日志会在storm ui上显示
* emit: emit一条消息
* metrics: rpc java metrics call，用作打点计数
* sync: 同步心跳，在处理一个非常耗时的任务时，需要在超时时间内调用这个函数，否则会被父进程判定为心跳超时强制kill

从ShellBolt到子进程的消息有两种类型，BoltMsg和List<?>，序列化消息的类由TOPOLOGY_MULTILANG_SERIALIZER配置指定，默认使用的是backtype.storm.multilang.JsonSerializer，如果需要自己实现序列化方法，要实现backtype.storm.multilang.ISerializer接口。

BoltMsg对应了storm.py中的Tuple实现，ShellBolt中会产生两种类型的BoltMsg，一种是在execute函数中，通过要执行的Tuple生成；另一种是检测到子进程心跳超时，发送的心跳探测包，这个包中的stream字段设置为_heartbeat，如果子进程存在，则会回复sync，否则ShellBolt会尝试kill掉子进程并退出。

List<?>消息只在可靠消息中生效，消息中记录的是emit返回的taskId，子进程会将这些ID放进pending_taskids的deque中，然后在调用storm.emit的时候顺序返回给用户。

ShellBolt和子进程的通信方式比较简单，但是这个地方经常会遇到一个问题，由于storm使用的是子进程的标准输出来进行通信，而迁移的业务代码很多时候都会有打印到标准输出的内容，这些不符合格式的消息在序列化的时候就会导致程序出错退出。这里有两种解决办法：

1. 自己实现Serializer。可以修改storm.py，在打印的消息前加入特定的前缀，Serializer只处理带有前缀的输出。

2. 重定向标准输出。实现起来比较简单粗暴，在storm.py文件中加入和修改下面的代码：

```python
// add
sys.stdout.flush()
sys.stderr.flush()
__pipe_fd = os.dup(sys.stdout.fileno())
__pipe_file = os.fdopen(__pipe_fd, 'w')
os.dup2(sys.stderr.fileno(), sys.stdout.fileno()

// modify
def sendMsgToParent(msg):
    print >>__pipe_file, json_encode(msg)
    print >>__pipe_file, "end"
    __pipe_file.flush()
```

ShellBolt是一个正常Blot的扩展实现，充当子进程和storm之间的消息转换中间层，其中，使用了ShellProcess来监视子进程状态，ShellBoltMessageQueue来存放接收到的消息，并且使用两个线程_readerThread和_writerThread来读写子进程标准输入输出，详细实现不再赘述。
