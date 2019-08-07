Kafka:一个分布式流处理平台

4个核心API：

    The Producer API 允许一个应用程序发布一串流式的数据到一个或者多个Kafka topic。 
    The Consumer API 允许一个应用程序订阅一个或多个 topic ，并且对发布给他们的流式数据进行处理。 
    The Streams API 允许一个应用程序作为一个流处理器，消费一个或者多个topic产生的输入流，然后生产一个输出流到一个或多个topic中
    去，在输入输出流中进行有效的转换。 
    The Connector API 允许构建并运行可重用的生产者或者消费者，将Kafka topics连接到已存在的应用程序或者数据系统。比如，连接到一
    个关系型数据库，捕捉表（table）的所有变更内容。
    
kafka相关概念解释

      0.message：消息，kafka存储和通信的基本单位
      1.producer：消息生产者，发布消息到 kafka 集群的终端或服务。
      2.broker：kafka 集群中包含的服务器。一个缓存的代理，可以理解成一个应用进程
      3.topic：每条发布到 kafka 集群的消息属于的类别，即 kafka 是面向 topic 的。消息存放的目录即主题。topic由一些Partition Logs
      (分区日志)组成。就像数据库的table一样
      4.partition：partition 是物理上的概念，每个 topic 包含一个或多个 partition。kafka 分配的单位是 partition。
      5.consumer：从 kafka 集群中消费消息的终端或服务。
      6.consumer group：high-level consumer API 中，每个 consumer 都属于一个 consumer group，每条消息只能被 consumer group 
      中的一个 Consumer 消费，但可以被多个 consumer group 消费。
      7.replica：partition 的副本，保障 partition 的高可用。
      8.leader：replica 中的一个角色， producer 和 consumer 只跟 leader 交互。
      9.follower：replica 中的一个角色，从 leader 中复制数据。
      10.controller：kafka 集群中的其中一个服务器，用来进行 leader election 以及 各种 failover。
      11.offset：位移量记录消息的位置信息

Kafka 通过 topic 对存储的流数据进行分类。生产者往topic里写消息，消费者从读消息。为了做到水平扩展，一个topic实际是由多个partition组成的，遇到瓶颈时，
可以通过增加partition的数量来进行横向扩容。单个partition内是保证消息有序。每新写一条消息，kafka就是在对应的文件append写，所以性能非常高。
![image](https://github.com/itsohorriblela/Hadoop-Diary/blob/master/images/kafkaconstruct.png)
    
    每个Partition中的消息都是有序的，生产的消息被不断追加到Partition log上，其中的每一个消息都被赋予了一个唯一的offset值。 
    Kafka集群会保存所有的消息，不管消息有没有被消费；我们可以设定消息的过期时间，只有过期的数据才会被自动清除以释放磁盘空间。比如我们
    设置消息过期时间为2天，那么这2天内的所有消息都会被保存到集群中，数据只有超过了两天才会被清除。 
    Kafka需要维持的元数据只有一个–消费消息在Partition中的offset值，Consumer每消费一个消息，offset就会加1。其实消息的状态完全是由
    Consumer控制的，Consumer可以跟踪和重设这个offset值，这样的话Consumer就可以读取任意位置的消息。 
    把消息日志以Partition的形式存放有多重考虑，第一，方便在集群中扩展，每个Partition可以通过调整以适应它所在的机器，而一个topic又可
    以有多个Partition组成，因此整个集群就可以适应任意大小的数据了；第二就是可以提高并发，因为可以以Partition为单位读写了。
    
   Producers往Brokers里面的指定Topic中写消息，Consumers从Brokers里面拉去指定Topic的消息，然后进行业务处理。
   图中有两个topic，topic 0有两个partition，topic 1有一个partition，三副本备份。可以看到consumer gourp 1中的consumer 2没有分到partition处理
![image](https://github.com/itsohorriblela/Hadoop-Diary/blob/master/images/kafkadataflow.png)
   关于broker、topics、partitions的一些元信息用zk来存，监控和路由啥的也都会用到zk。
   
每条记录指定一个对应的topic和partition（可选），包含一个key，一个value和一个timestamp（时间戳）。 先序列化，然后按照topic和partition，放进对应的发送
队列中。kafka produce都是批量请求，会积攒一批，然后一起发送，不是调send()就进行立刻进行网络发包。
![image](https://github.com/itsohorriblela/Hadoop-Diary/blob/master/images/kafkaproductflow.png)

partition：

    Kafka中的topic是以partition的形式存放的，每一个topic都可以设置它的partition数量，Partition的数量决定了组成topic的log的数量。
    Producer在生产数据时，会按照一定规则（这个规则是可以自定义的）把消息发布到topic的各个partition中。不过只有一个partition的副本
    会被选举成leader作为读写用。 一个partition只能被一个消费者消费（一个消费者可以同时消费多个partition），因此，如果设置的partition
    的数量小于consumer的数量，就会有消费者消费不到数据。所以，推荐partition的数量一定要大于同时运行的consumer的数量。另外一方面，
    建议partition的数量大于集群broker的数量，这样leader partition就可以均匀的分布在各个broker中，最终使得集群负载均衡。

producer：

    Producers直接发送消息到broker上的leader partition，不需要经过任何中介一系列的路由转发。kafka集群中的每个broker都可以响应producer
    的请求，并返回topic的一些元信息，这些元信息包括哪些机器是存活的，topic的leader partition都在哪，现阶段哪些leader partition是可以
    直接被访问的。 以Batch的方式推送数据可以极大的提高处理效率，kafka Producer 可以将消息在内存中累计到一定数量后作为一个batch发送请求。
    Batch的数量大小可以通过Producer的参数控制，参数值可以设置为累计的消息的数量（如500条）、累计的时间间隔（如100ms）或者累计的数据大小(64KB)。
    通过增加batch的大小，可以减少网络请求和磁盘IO的次数，
    acks：producer要求leader partition 收到确认的副本个数
    acks=0:producer不会等待broker的响应,producer无法知道消息是否发送成功，这样有可能会导致数据丢失，但会得到最大的系统吞吐量
    acks=1:producer会在leader partition收到消息时得到broker的一个确认
    acks--1:producer会在所有备份的partition收到消息时得到broker的确认，这个设置可以得到最高的可靠性保证。 
    Kafka 消息有一个定长的header和变长的字节数组组成。因为kafka消息支持字节数组，也就使得kafka可以支持任何用户自定义的序列号
    格式或者其它已有的格式如Apache Avro、protobuf等。
 
 consumers
 
    在kafka中，当前读到消息的offset值是由consumer来维护的，因此，consumer可以自己决定如何读取kafka中的数据。比如，consumer可以通过重设offset
    值来重新消费已消费过的数据。不管有没有被消费，kafka会保存数据一段时间，这个时间周期是可配置的，只有到了过期时间，kafka才会删除这些数据。
    每条消息再一个消费组里只能消费一次，不同消费组可以重复消费一条消息
 
 消息可靠性
 
    当一个消息被发送后，Producer会等待broker成功接收到消息的反馈（可通过参数控制等待时间），如果消息在途中丢失或是其中一个broker挂掉，Producer
    会重新发送当Consumer收到了消息，但却在处理过程中挂掉，Consumer可以通过这个offset值重新找到上一个消息再进行处理。
    
备份机制

