# BitCask

<img src=".\asset\image-20211018011541860.png" alt="image-20211018011541860" style="zoom:80%;" />

## KV 存储背景

在kv存储上,有三种主流的数据结构设计

- B+ Tree
  - BerkeleyDB at Oracal, WiredTiger in MongoDB
- Log-Structured Merge Tree(LSM-tree)
  - in-disk sorted-log-structured data
  - LevelDB and BigTable at Google, RocksDB at Facebook, Cassandra, HBase and Accumulo at Apache, Dynamo at Amazon.
- Log-Structured Hash Table
  - in-memory hash table, in-disk log-structured data
  - BitCask at Riak, Sparkey at Spotify, FASTER at Microsoft

### 什么叫LSM?

Log-Structured 指将kv-item以append的方式写入文件,就像日志一样,按照时序追加写,以此来获得较大的写吞吐量

Merge 指通过merge多个文件或逻辑块来删除掉过时数据的过程

## Goals

一个local k/v store 需要解决的几个问题:

• low latency per item read or written
• high throughput, especially when writing an incoming stream of random items
• ability to handle datasets much larger than RAM w/o degradation
• crash friendliness, both in terms of fast recovery and not losing data
• ease of backup and restore
• a relatively simple, understandable (and thus supportable) code structure and data format
• predictable behavior under heavy access load or large volume

**Achieving some of these is easy. Achieving them all is less so.**

## 适用场景

Suitable for write-heavy application with ample memory, and no range reads

## 实现

### The Log-Structured FS

bitcask 将kv item数据写入文件系统, 在任何一个时刻,只有一个文件是打开的(称为active file), 如果它的文件大小到达一个阈值,他就会被关闭,然后一个新的active file被打开.

一个被关闭的文件,无论是因为主动关闭(因文件大小到达阈值),还是由于server意外退出,它永远不会被再打开,其数据被认为是不可变的(immutable)

<img src=".\asset\image-20211017173114988.png" alt="image-20211017173114988" style="zoom:75%;" />

### 顺序写

Active file只会以append的方式写入,这是因为磁盘顺序写不需要寻道时间

### kv落盘结构

bitcask设计的在磁盘上的kv item的结构是非常简单的

- crc用于校验,防止bit位的错误
- tstamp指示写入时间
- key size, value size 是定长的,用于表示不定长的key,value的大小

<img src="C:\Users\salvare000\AppData\Roaming\Typora\typora-user-images\image-20211017173315107.png" alt="image-20211017173315107" style="zoom:80%;" />

<img src=".\asset\image-20211017173619360.png" alt="image-20211017173619360" style="zoom:80%;" />



**Thus, a Bitcask data file is nothing more than a linear sequence of these entries**

### 内存中的索引

bitcask中,位于内存中的数据结构称为`keydir`,起索引的作用,这个数据结构本身是一个扁平化的hash表(非树状或层级表)

<img src=".\asset\image-20211017174609807.png" alt="image-20211017174609807" style="zoom:80%;" />

<img src="http://my.huhoo.net/study/bitcask.jpg" alt="存储引擎简介- My Study" style="zoom:80%;" />

### CRUD

- create: 创建一条新记录,很自然的使用append方式追加到active file
- read: 通过内存中的hash表,通过key索引得到该key对应的value在磁盘中的位置
- update: 在任何类LSM树的结构中,update等同于create一条新记录
- remove: 在任何类LSM树的结构中,update等同于create一条新记录(比如value_size设置为-1或0表示删除)

每当append active fiel之后,更新内存中的索引hash表,这是一个事务,只有文件append和内存索引更新后,一个kv才算创建/删除/更新成功

> 以update为例,假如更新数据1:"old data" -> 1:"new data",那么首先{1: "new data"}会以append的形式写入active file,然后内存索引keydir会更新key的值,将原来的file_id,value_sz,value_pos,ts原地(in-place)更新为新的file_id,value_sz,value_pos,ts.

>  内存是随机读写的,而磁盘随机读写有较大消耗,所以采用追加写随机读来代替. 由于对于存储引擎来说,key的到来是一个随机的输入流,显然读不可能优化成顺序读,所以读必然需要一次或多次磁盘读,在bitcask中,由于在内存中维护了索引,所以读也至多只需要一次寻道时间(在某些树状索引中,如果索引不在内存中,需要读多次索引(取决于树的高度)得到value的位置,每读一次就是一次磁盘寻道)

很显然,旧记录还存在,这会造成磁盘空间的放大,因此需要merge的过程



### Merge

由于类lsm树的存储方式,旧的数据是不care,每次的写入,无论是新建还是修改还是删除,都只是统一的以追加的方式写入文件

虽然我们认为外部存储的空间的廉价的,短时间内可以不管,但是不能一直不去处理这些已经没有任何意义的旧数据

在后台,会定时的,或以其他某种方式触发进程用于compaction,或者叫做merge. merge进程遍历读取所有的非active的文件,处理,然后生成一系列只有有效数据(或者说最新数据)的文件

处理的过程也是简单的,每当遇到相同的key,我们比较它们的时间戳,有较新的时间戳的数据被认为是有效的数据. (有点类似mvcc,每个时间戳是一个version)

至于如何在较小的内存中处理较大的外部数据,可以参考外排序,只是这里我们的计算不是排序,而是去重.(在一些提供按key_prefix scan功能的kv存储引擎中,比如rocksdb,merge的过程就不仅仅是去重,也包括按key进行排序)

#### hint-file

多个文件被merge后,生成多个merged data file,这些data file会生成相伴随的hint-file,这个hint-file用于存储这个相对应的data file的索引数据

生成的hint-file的好处是可以让其他读进程快速构建索引,也可以在服务启动或恢复时快速恢复索引,只需读完hint-file就可以获得索引而不需要遍历数据文件

## 结论

1. low latency per item read or written

   read: 通过内存中的索引,一个随机读只需要一次磁盘寻道时间

   write: 通过磁盘顺序写,不需要磁盘寻道时间

2.  high throughput, especially when writing an incoming stream of random items

   高写吞吐量,因为以类LSM-Tree的方式组织磁盘数据文件,只需追加写

   一个搭载磁盘的笔记本上,每秒可以达到5k-6k的写

3.  ability to handle datasets much larger than RAM w/o degradation

   因为内存中只存储索引,而索引的item本身是定长且较小的

4. crash friendliness, both in terms of fast recovery and not losing data

   数据文件本身就是类似write-ahead log, 只要写入了文件就被持久化了,恢复只需读取hint-file形成索引即可

5. ease of backup and restore

   备份只需复制数据文件和hint-file即可(也就是复制文件夹)

   恢复只需把备份的文件拷贝到文件夹即可

6. a relatively simple, understandable (and thus supportable) code structure and data format

   数据结构本身是简洁明了的

## 缺点

1. 要在内存中维护所有key的索引,需要有足够大的内存
   1. 但是可以通过分片sharding的方式做水平拓展
2. 类LSM的通病: 写放大

## Ref

[J. Sheehy and D. Smith. Bitcask: A Log-Structured Hash Table for
Fast Key/Value Data. Basho White Paper, 2010.](https://riak.com/assets/bitcask-intro.pdf)

[Idreos, Stratos et al. “Learning Key-Value Store Design.” (2019)](https://arxiv.org/pdf/1907.05443.pdf)

[[FAST '16] *WiscKey*, *Separating Keys* from Values in SSD-conscious Storage](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf)
