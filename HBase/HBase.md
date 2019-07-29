HBase：
在hdfs上开发的面对列的分布式数据库

1.特点：
    1 大：一个表可以有上亿行，上百万列
    2 面向列:面向列(族)的存储和权限控制，列(族)独立检索。
    3 稀疏:对于为空(null)的列，并不占用存储空间，因此，表可以设计的非常稀疏
    4 无模式：每行都有一个可排序的主键和任意多的列，列可以根据需要动态的增加，同一张表中不同的行可以有截然不同的列；
    5 数据多版本：每个单元中的数据可以有多个版本，默认情况下版本号自动分配，是单元格插入时的时间戳；
    6 数据类型单一：Hbase中的数据都是字符串，没有类型。

2.基本概念：
    RowKey：是Byte array，是表中每条记录的“主键”，方便快速查找，Rowkey的设计非常重要。
    Column Family：列族，拥有一个名称(string)，包含一个或者多个相关列。hbase表中的每个列，都归属与某个列族。列族是表的chema的一部分(而列不是)，必须在
    使用表之前定义。
    Column：属于某一个columnfamily。列名都以列族作为前缀：FamilyName:columnName，每条记录可动态添加
    Version Number：类型为Long，默认值是系统时间戳，时间戳可以由hbase(在数据写入时自动 )赋值，此时时间戳是精确到毫秒的当前系统时间。时间戳也可以由客户
    显式赋值。HBase中通过row和columns确定的为一个存贮单元称为cell，每个 cell都保存着同一份数据的多个版本，版本通过时间戳来索引。
    Value(Cell)：Byte array.由{row key, column( =<family> + <label>), version} 唯一确定的单元。cell中的数据是没有类型的，全部是字节码形式存贮。

3.物理存储：
    Table中的所有行都按照row key的字典序排列，Table 在行的方向上分割为多个Hregion。region按大小分割的，每个表一开始只有一个region，随着数据不断插入表
    ，region不断增大，当增大到一个阀值的时候，Hregion就会等分会两个新的Hregion。当table中的行不断增多，就会有越来越多的Hregion。Hregion是Hbase中分布
    式存储和负载均衡的最小单元。最小单元就表示不同的Hregion可以分布在不同的HRegion server上。但一个Hregion是不会拆分到多个server上的。
![image](https://github.com/itsohorriblela/Hadoop-Diary/blob/master/images/hregion.png)
     HRegion虽然是分布式存储的最小单元，但并不是存储的最小单元。事实上，HRegion由一个或者多个Store组成，每个store保存一个columns family，每个Store
     又由一个memStore和0至多个StoreFile组成，StoreFile以HFile格式保存在HDFS上。
![image](https://github.com/itsohorriblela/Hadoop-Diary/blob/master/images/strore.png)

一个master节点协调管理一个或多个regionserver从属机
![image](https://github.com/itsohorriblela/Hadoop-Diary/blob/master/images/hbasemembers.png)

4.系统架构：
    Client：包含访问HBase的接口，并维护cache来加快对HBase的访问，比如region的位置信息
    Master：为Region server分配region，负责Region server的负载均衡，发现失效的Region server并重新分配其上的region，管理用户对table的增删改查操作
    Region Server：Regionserver维护region，处理对这些region的IO请求，Regionserver负责切分在运行过程中变得过大的region
    Zookeeper
    1 保证任何时候，集群中只有一个master，Master与RegionServers 启动时会向ZooKeeper注册，Zookeeper的引入使得Master不再是单点故障
    2 存贮所有Region的寻址入口。
    3 实时监控Region Server的状态，将Region server的上线和下线信息实时通知给Master
    4 存储Hbase的schema,包括有哪些table，每个table有哪些column family
    
WAL（Write-Ahead-Log）：用于数据的容错和恢复。Hlog记录数据的所有变更,一旦数据修改，就可以从log中进行恢复。每个Region Server维护一个Hlog,而不是每个
Region一个。

5.流程

6.容错性
