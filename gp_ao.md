# Greenplum AO 表支不支持 index scan

https://github.com/greenplum-db/gpdb/pull/16231 
在这个pr之前，在 Greenplum 的 AO 表上只支持使用 sequence scan , bitmap scan 和 index-only，不支持index scan. 

## 让 AO 表支持 pgvector index scan

该pr主要是为了使 GP AO 表支持 pgvector 的 index  scan (IVFFLAT & HNSW)

提出这个 pr 的原因 是 GP 想支持 pgvector，但是 pgvector 里面的两个 index : ivfflat 和 hnsw 都不支持 bitmap scan，AO 表又不支持 Index scan , 也就是没有办法在向量数据上使用上述两种索引，只能进行顺序扫描，这样 pgvector 就没用了。

解释一下为什么 pgvector 的 index 不支持 bitmap scan。 pgvctor 是专门处理 vector 数据的插件，包括 vector 新类型的引入，及对应的索引用来支持 ANN SEARCH, 即查找 n 个最近的邻居，并按照距离排序返回。

```
SELECT * FROM t ORDER BY val <-> '[3,3,3]' limit 10;
```
上面例子是最常见的查询向量的语句。
-	pgvector index 返回的数据是排序好的
-	需要 index 返回的数据量不大

根据上述描述，可见现阶段没有必要让 pgvector 支持 bitmap index。因为如果使用 bitmap index scan 还需要对数据进行重新排序。
bitmap scan 一般用在多个单列过滤条件的情况下。

ps: 目前 pgvertor 对于属性 filter 并不支持. https://github.com/pgvector/pgvector/issues/259 

## 为什么AO表不支持index scan
https://github.com/greenplum-db/gpdb/pull/16231， 
在这个pr（支持pgvector index之前），其实已经有其它的pr 让AO表支持index scan，但是最终close了。
https://github.com/greenplum-db/gpdb/pull/5450 

根本原因就是 AO 表的随机读性能差
- AO 表的文件块大小是变长的，没办法根据 TID 定位到具体的文件块物理位置。为了支持随机读和索引，AO 表引入了 blockDir 存储 TID 到文件物理位置的映射，会有一定的性能损失。

- AO表没有buffer pool, 每次读一个 block 的时候都需要读文件再重新解压，性能很差。

但是Index scan 会随机读 AO 表。因此 AO 表只支持 bitmap scan 和 Index only scan, 避免随机读。

