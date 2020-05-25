## 1 场景介绍

​	ES 集群架构 13 个节点，索引根据通道不同共 20+索引，根据日期，每日递增 20+，索引：10 分片，每日递增 300w+数据，每个通道每天索引大小控制：150GB 之内。

### 1.1 设计阶段调优

- 仅针对需要分词的字段，合理的设置分词器；
- Mapping 阶段充分结合各个字段的属性，是否需要检索、是否需要存储等；
- 每天凌晨定时对索引做 force_merge 操作，以释放空间；

### 1.2 写入调优

- 写入前副本数设置为 0；

- 写入前关闭 refresh_interval 设置为-1，禁用刷新机制；

- 写入过程中：采取 bulk 批量写入；

- 写入后恢复副本数和刷新间隔；

- 尽量使用自动生成的 id。

### 1.3 查询调优

- 禁用批量 terms（成百上千的场景）
- 充分利用倒排索引机制，能 keyword 类型尽量 keyword；
- 数据量大时候，可以先基于时间敲定索引再检索；

### 1.4 其他调优

​	部署调优，业务调优等等。

​	比如在ES只存放关键必要字段，Hbase存储完整的数据。根据条件查询出ID之后，再去Hbase中查询完成的数据等等。

## 2 倒排索引

​	通过分词策略，形成分词和文章ID集合的映射关系表，可以再O(1)的时间复杂度内根据分词找到对应的文章，检索效率特别高。

​	倒排索引的底层实现是基于：**FST**（`Finite State Transducer`）数据结构。

​	lucene 从 4+版本后开始大量使用的数据结构是` FST`。`FST` 有两个优点：

- 空间占用小。通过对词典中单词前缀和后缀的重复利用，压缩了存储空间；

- 查询速度快。O(len(str))的查询时间复杂度。

|  id  | name | sex  |
| :--: | :--: | :--: |
|  1   |  A   |  男  |
|  2   |  B   |  女  |
|  3   |  C   |  男  |

这张表的倒排索引就是：

name:

| A    | 1    |
| ---- | ---- |
| B    | 2    |
| C    | 3    |

sex:

|  男  | [1,3] |
| :--: | :---: |
|  女  |  [2]  |

​	**倒排索引是per field的，一个字段有一个自己的倒排索引**。男、女 这些叫做 term，而[1,3]就是posting list。Posting list就是一个int的数组，存储了所有符合某个term的文档id。

​	如果要搜索的是一堆无序的词：Carla,Sara,Elin,Ada,Patty,Kate,Selena。那么如果要找出某一个term（项）一定很慢，因为term是无序的，需要全部遍历一遍才能找到term。

​	如果词是有序的，那么我们可以根据二分查找，比全遍历更快地找出目标的term。这个就是term directonary，在log(n)的时间复杂度内完成检索，也就是进行log(n)次磁盘读取。但是磁盘随机读操作是非常耗时的，一般一次随机访问大概需要10ms。所以应该尽量少得读磁盘，有必要把一些数据缓存到内存中。但是整个term dictionary本身又太大了，无法完整地放到内存里。于是就有了term index。term index有点像一本字典的大的章节表。

​	如果所有的term都是英文字符的话，可能这个term index就真的是26个英文字符表构成的了。但是实际的情况是，term未必都是英文字符，term可以是任意的byte数组。而且26个英文字符也未必是每一个字符都有均等的term，比如x字符开头的term可能一个都没有，而s开头的term又特别多。实际的term index是一棵trie 树：

​	这棵树不会包含所有的term，它包含的是term的一些前缀。通过term index可以快速地定位到term dictionary的某个offset，然后从这个位置再往后顺序查找。再加上一些压缩技术（搜索 Lucene Finite State Transducers） term index 的尺寸可以只有所有term的尺寸的几十分之一，使得用内存缓存整个term index变成可能。整体上来说就是这样的效果。

​	Elasticsearch比mysql快的原因是：Mysql只有term dictionary这一层，是以b-tree排序的方式存储在磁盘上的。检索一个term需要若干次的random access的磁盘操作。而Lucene在term dictionary的基础上添加了term index来加速检索，term index以树的形式缓存在内存中。从term index查到对应的term dictionary的block位置之后，再去磁盘上找term，大大减少了磁盘的random access次数。

​	额外值得一提的两点是：term index在内存中是以FST（finite state transducers）的形式保存的，其特点是非常节省内存。Term dictionary在磁盘上是以分block的方式保存的，一个block内部利用公共前缀压缩，比如都是Ab开头的单词就可以把Ab省去。这样term dictionary可以比b-tree更节约磁盘空间。

## 3 ES索引数据多了、如果调优

​	索引数据的规划，应在前期做好规划，正所谓“设计先行，编码在后”，这样才能有效的避免突如其来的数据激增导致集群处理能力不足引发的线上客户检索或者其他业务受到影响。