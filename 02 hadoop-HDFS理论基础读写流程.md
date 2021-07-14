# 02 hadoop-HDFS理论基础读写流程

问题：分布式文件系统那么多，

为什么hadoop项目中还要开发一个hdfs文件系统？（一直带着这个问题学习hadoop）

## 架构设计

- HDFS是一个主从（Master/Slaves）架构
- 由一个NameNode和一些DataNode组成
- 面向文件包含：文件数据（data）和文件元数据（metadata）
- NameNode负责存储和管理文件元数据，并维护了一个层次型的文件目录树
- DataNode负责存储文件数据（block块），并提供block的读写。
- DataNode与NameNode维持心跳，并汇报自己持有的block信息
- Client和NameNode交互文件元数据和DataNode交互文件block数据

## 架构图



角色即JVM进程![image-20210703192209125](https://raw.githubusercontent.com/Berserker-DHC/PicStore/master/img/20210704130410.png)

客户端和namenode交互元数据

名称，副本的数量

![image-20210704132228480](https://raw.githubusercontent.com/Berserker-DHC/PicStore/master/img/20210704132228.png)



## 角色功能

NameNode

- 完全基于内存（为了快速提供地址）存储文件元数据、目录结构、文件block的映射
  - 第一不可靠
  - 第二大小有限制
    - redis
    - datasearch
    - HBase
    - 要持久化方案 ->
- 需要持久化方案保证数据可靠性
- 提供**副本**放置策略

DataNode

- 基于进程所在的操作系统的本地磁盘，存储block（文件的形式）
- 并保存block的校验和（MD5）数据保证block的可靠性
- 与NameNode保持心跳，汇报block列表的状态



## 元数据持久化

数据持久化的方案

**第一个方案**

日志文件：记录实时发生的增删改的操作 mkdir /abc append 文本文件

- 完整性比较好
- 加载恢复数据：慢/占空间     <    比如说，4G内存 运行了10年 日志很大
- 5年恢复
- 内存会不会溢出
- 内存不会溢出
- 内存边创建边删除
- 日志可能几百个T，内存还是不变



**第二个方案**

镜像，快照，dump，db，序列化（字节数组）

- 间隔（小时，天，10分钟，1分钟，5秒），内存全量数据基于某一个时间点做的向磁盘的溢出
- 考虑到IO慢的时间消耗大，不能频繁进行，所以最好按天，按小时

请问999999999（9个9）占用多大的磁盘空间

- 以什么文件编码：
- txt byte 9个字节
- java中的 int a = 999999999，4个字节（内存编码都是二进制文件）
- 加载会比较慢，
- 快照的方式，
  - 优点：恢复速度一定快过日志文件，
  - 缺点：因为是间隔的，容易丢失一部分数据

HDFS

- EditsLog：日志

  - 体积小，记录少：必然有优势

- FsImage：镜像、快照

  - 如果能提高滚动更新频率

- 最近时间点的FsImage + 增量的EditsLog

  - 现在10点关机，需要恢复数据

  - FI9点 + 9点到10点的增量的EL

    - 1. 加载FsImage

      2. 加载EL

      3. 内存就得到了关机前的log

**总结**：

- 任何对文件系统元数据产生修改的操作，NameNode都会使用一种称为EditLog的事务日志记录下来
- 使用FsImage存储内存所有的元数据状态
- 使用本地磁盘保存EditLog和FsImage
- EditLog具有完整性，数据丢失少，但恢复速度慢，并由体积膨胀的风险
- FsImage具有恢复速度快，体积与内存数据相当，但不能实时保存，数据丢失多
- NameNode使用了FsImage + EditLog整合的方案
  - 滚动将增量的EditLog更新到FsImage，以保证更近时点的FsImage和更小体积的EditLog

**问题**：

- 那么，FI时点是如何滚动更新的？

**答**：

- 由NameNode8点溢写一次，9点溢写一次
- NameNode：第一次开机时，只写一次FI，假设8点，到9点时，EL记录的是8~9点的日志，
- 只需要将8~9点的日志的记录，更新到8点的FI中，FI的数据时点就变成了9点。
- 寻求另外一台机器做合并，NameNode不用做



## 安全模式

- HDFS搭建时会格式化，格式化操作会产生一个空的FsImage
- 当NameNode启动时，从硬盘中读取Editlog和FsImage
- 将所有EditLog中的事务作用在内存中的FsImage上
- 并将这个新版本中的FsImage从内存中保存到本地磁盘上
- 然后删除旧的EditLog，因为这个旧的EditLog的事务都已经作用在FsImage上了

## 知识点

- NameNode存元数据：文件属性 / 每个块存在哪个DataNode上
- 在持久化的时候：文件属性会持久化，但是文件的每一个块不会持久化
- 在恢复的时候，NameNode会丢失块的位置信息

## HDFS中的SNN

SecondaryNameNode(SNN)

- 在非HA模式下，SNN一般是独立的节点（JVM的一个进程），周期完成对NN的Editlog向FsImage合并，减少EditLog大小，减少NN启动事件。

- 根据配置文件设置的时间间隔fs.checkpoint.period 默认3600秒

- 根据配置文件设置的EditsLog大小，fs.checkpoint.size规定edits文件的最大默认值是64MB

  ![image-20210714144554793](https://raw.githubusercontent.com/Berserker-DHC/PicStore/master/img/20210714144602.png)

## Block的副本放置策略

- 第一个副本：放置在上传文件的DN；
  如果是集群外提交，则随机挑选一台磁盘不太满，CPU不太忙的节点。
- 第二个副本：放置在与第一个副本不同的机架的节点上。
- 第三个副本：与第二个副本相同机架的节点。
- 更多副本：随机节点。

## HDFS写流程

![image-20210714152713045](https://raw.githubusercontent.com/Berserker-DHC/PicStore/master/img/20210714152713.png)

## HDFS写流程

- Client和NN连接创建文件元数据
- NN判定元数据是否有效
- NN处发副本放置策略，返回一个有序的DN列表
- Client和DN建立Pipeline连接
- Client将块切分成packet（64KB），并使用chunk（512B）+chunksum（4B）填充
- Client将packet放入发送队列dataqueue中，并向第一个DN发送
- 第一个DN收到packet后本地保存并发送给第二个DN
- 第二个DN收到packet后本地保存并发送给第三个DN
- 这一个过程中，上游节点同时发送下一个packet
- 生活中类比工厂的流水线，结论：流失，其实也是变种的并行计算
- HDFS使用这种传输方式，副本数对于client也是透明的
- 当block传输完成，DN们各自向NN汇报，同时client继续传输下一个block
- 所以，client的传输和block的汇报也是并行的

## HDFS读流程

- ![image-20210714175215398](https://raw.githubusercontent.com/Berserker-DHC/PicStore/master/img/20210714175215.png)

- 为了降低整体的带宽消耗和读取延时，HDFS会尽量让读取程序读取离它最近的副本
- 如果在读取程序的同一个机架上有一个副本，那么就读取该副本
- 如果一个HDFS集群跨越多个数据中心，那么客户端也将首先读本地数据中心的副本
- 语义：下载一个文件：
  - Client和NN交互文件元数据获取fileBlockLocation
  - NN会按距离策略排序返回
  - Client尝试下载block并校验数据完整性
- 语义：下载一个文件其实是获取文件的所有的block元数据，那么子集获取某些block应该成立
  - HDFS支持client给出文件的offset自定义连接哪些block的DN，自定义获取数据
  - 这个是支持计算层的分治、并行计算的核心
