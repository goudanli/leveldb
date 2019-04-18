### SSTable
SSTable格式如下：


1)SSTable中的k/v数据对是有序存储的，被划分到连续的Data Block中,Data Block的存储格式代码在block_builder.cc中

2)紧跟在Data Block之后的是Meta Block，其格式代码也在block_builder.cc中；Meta Block存储的是Filter信息，比如Bloom过滤器，用于快速定位key是否在data block中。

3）Meta Index Block是对Meta Block的索引，它只有一条记录，key是meta index的名字（也就是Filter的名字），value为指向meta index的BlockHandle；

BlockHandle是一个结构体，成员offset_是Block在文件中的偏移，成员size_是block的大小；

4）Index block是对Data Block的索引，对于其中的每个记录，其key >=Data Block最后一条记录的key，同时<其后Data Block的第一条记录的key；value是指向data index的BlockHandle；




#### Data Block 组成
Data Block 是具体的k/v数据存储区域，也存储meta,索引信息，他们都是以Data Block的形式存储的，Block组成 如图：
|unsigned char[]|1 byte| 4 bytes|
| :----------:  |:---: | :----: |
| Block Data    | type | crc32  |

- 类型type指明使用的是哪种压缩方式，当前支持none和snappy压缩。
- 虽然block有好几种，但是Block Data都是有序的k/v对，因此写入、读取BlockData的接口都是统一的，对于Block Data的管理也都是相同的

##### Block data 
格式如下：

```
shared_bytes    varint32  //共享key的长度(前缀)
unshared_bytes  varint32  //非共享key长度（前缀之后的字符串）
value_length    varint32  //值长度
key_delta       char[unshared_bytes] //前缀之后的字符串
value           char[value_length]

Block的结尾段格式是：
restarts       uint32[num_restarts] //重启点集合
num_restarts  uint32 //重启点个数

```
BlockBuilder对key的存储是前缀压缩的，对于有序的字符串来讲，这能极大的减少存储空间。但是却增加了查找的时间复杂度，为了兼顾查找效率，每隔K个key，leveldb就不使用前缀压缩，而是存储整个key，这就是重启点（restartpoint）。

在构建Block data时，有参数Options::block_restart_interval定每隔几个key就直接存储一个重启点key。

Block在结尾记录所有重启点的偏移，可以二分查找指定的key。Value直接存储在key的后面，无压缩

总体来看Block data可分为k/v存储区和后面的重启点存储区两部分，其中k/v的存储格式如前面所讲，可看做4部分：

前缀压缩的key长度信息 + value长度 + key前缀之后的字符串+ value

最后一个4byte为重启点的个数


block_builder.cc 主要实现了Block data的构建过程，Block data内部有很多k，v数据对
block.cc  主要实现了Block data的内部遍历



