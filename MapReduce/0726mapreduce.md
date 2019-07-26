MapReduce:用于大规模数据集并行运算的编程模型

Hadoop将mapreduce的输入数据划分成小数据块，称为input split，并为每个split构建map任务。因此，单个split越小，split的总数便越多，因为每个节点的处理速度
不一样，所以更多的split意味着更容易实现负载的平衡。另一方面，如果split分得太小，那么管理split、构建map任务所占时间的比例将会更大。因此一个合理的split
的大小约等于一个block的大小————可以略小，不能过大，因为如果一个split跨越两个block，但是每个节点上不一定会同时存在这两个block,便导致了节点与节点之间的
网络传输。

map任务得到的结果是一个中间值，存储在本地磁盘中，然后所有map任务的输出一起输入到reduce的节点上进行reduce操作，reduce任务的输出最终存入hdfs中。map与
redeuce之间的数据流称为shuffle（混洗），因为每个reduce的输入都来自很多map的输出。
![image](https://github.com/itsohorriblela/Hadoop-Diary/tree/master/images/shuffle.png)

