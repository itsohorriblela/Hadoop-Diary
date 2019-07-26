Hadoop 一套开源软件平台，对数据进行分布式处理。

核心组件：hdfs：分布式文件系统

mapreduce：分布式运算编程框架 

yarn：为mapreduce程序分配运算硬件资源调度的系统，允许任何一个分布式程序基于Hadoop集群的数据运行

zookeeper：分布式应用程序协调服务

hbase：分布式非关系型数据库，一种使用hdfs做底层存储的键值存储模型

hdfs

1.适合部署在廉价的机器上的高度容错性的系统：硬件错误是常态而不是异常。

2.更多的考虑到了数据批处理，而不是用户交互处理。

3.“一次写入多次读取”的文件访问模型。

4.将计算移动到数据附近，比之将数据移动到应用所在显然更好。HDFS为应用提供了将它们自己移动到数据附近的接口。

HDFS采用master/slave架构，一个HDFS集群是由一个Namenode和一定数目的Datanodes组成。

namenode：一个中心服务器，负责管理文件系统的名字空间(namespace)以及客户端对文件的访问。执行文件系统的namespace操作，比如打开、关闭、重命名文件或目
录。它也负责确定blocks到具体Datanode节点的映射。

datanode：一个节点一个，负责管理它所在节点上的存储，负责处理文件系统客户端的读写请求。在Namenode的统一调度下进行blocks的创建、删除和复制。

blocks：一个文件被分成一个或多个数据块（blocks），这些块存储在一组Datanode上。HDFS被设计成支持大文件，适用HDFS的是那些需要处理大规模的数据集的应用。
这些应用都是只写入数据一次，但却读取一次或多次，并且读取速度应能满足流式读取的需要。HDFS支持文件的“一次写入多次读取”语义。一个典型的数据块大小64MB。
因而，HDFS中的文件总是按照64M被切分成不同的块，每个块尽可能地存储于不同的Datanode中。在开始阶段文件数据会先缓存在本地的一个临时文件中，当临时文件的数
据量超过一个block的大小时才会上传到指定的datanode上。

副本系数：文件副本的数目称为文件的副本系数，这个信息也是由Namenode保存的。

hdfs的存储策略

  hdfs如何保证存储的可靠性？HDFS采用一种称为机架感知(rack-aware)的策略来改进数据的可靠性、可用性和网络带宽的利用率。为避免数据丢失，最常见的方法便复
制。通常我们把副本系数设为3，一个副本存放在同一个节点上，一个副本存放在同一机架的不同节点上，一个副本放在不同的机架上。这种策略减少了机架间的数据输，
提高了写操作的效率。机架的错误远远比节点的错误少，所以这个策略不会影响到数据的可靠性和可用性。因为数据块只放在两个不同的机架上，所以此策略减少了读取数
据时需要的网络传输总带宽。

Namenode全权管理数据块的复制，它周期性地从集群中的每个Datanode接收心跳信号和块状态报告(Blockreport)。接收到心跳信号意味着该Datanode节点工作正常。每
个Datanode节点周期性地向Namenode发送心跳信号，近期不再发送心跳信号的节点将被标注为宕机，某些datanode的宕机可能会导致一些数据块的副本系数低于正常值，
Namenode不断地检测这些需要复制的数据块，一旦发现就启动复制操作。

元数据

对于任何对文件系统元数据产生修改的操作，Namenode都会使用一种称为EditLog的事务日志记录下来。Editlog被存储在本地操作系统的文件系统中。整个文件系统的
namespace，包括数据块到文件的映射、文件的属性等，都存储在一个称为FsImage的文件中，这个文件也是放在Namenode所在的本地文件系统上。

检查点（checkpoint）：当Namenode启动时，它从硬盘中读取Editlog和FsImage，将所有Editlog中的事务作用在内存中的FsImage上，并将这个新版本的FsImage从内
存中保存到本地磁盘上，然后删除旧的Editlog。

块状态报告：当一个Datanode启动时，它会扫描本地文件系统，产生一个这些本地文件对应的所有HDFS数据块的列表，然后作为报告发送到Namenode。

流水线复制；当本地临时文件积累到一个block大小时开始向第一个datanode传送数据，第一个datanode在接受数据的同时传送给第二个datanode，以此类推，直到最后
一个datanode存储完成。
![image](https://github.com/itsohorriblela/Hadoop-Diary/upload/master/images/clientwriteonHDFS.png)
![image](https://github.com/itsohorriblela/Hadoop-Diary/blob/master/images/clientreadfromHDFS.png)
HDFS总览
![image](https://github.com/itsohorriblela/Hadoop-Diary/tree/master/images/HDFS.png)
