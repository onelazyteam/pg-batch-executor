# 为什么要去实现一个批处理的执行器？

刚开始接触openGauss的时候，发现它有个opfusion（也叫SQLByPass）的执行器，开始看这个代码的时候觉得这个设计挺有新意的，摒弃掉了火山模型执行器调用层次过深带来的性能上损耗，通过直接裸调用存储层的接口来优化简单DML语句的性能，后来一次偶然机会和几个朋友聊这个设计，有个人就提了一个疑惑，这个设计唯一的优势就是减少了多层函数调用的开销，而且只能支持简单语句场景，为什么不直接设计一个批处理模型呢？我当时被问住了，想了一下，确实如此，批处理模型也可以实现相同的效果，而且是个通用型的模型，不受场景限制，openGauss虽然已经支持了向量化引擎，并支持了行转列的算子，也就是即使是行存表也可以通过一个行转列算子来利用向量化走批处理模型，**那为什么不直接使用向量化引擎呢？**行转列走向量化有如下几个问题：

1. 当前的行转列是加了两个Adapter算子，这会多出不必要的开销，这个开销是相当大的，操作等于对每行每列数据都需要进行一次重新读出组装，所以行转列是一比很大的开销且输出的时候是行存，所以最后输出给客户端前还要做一次转行操作

   再有下面的Vector Adapter并没有和SeqScan做融合，等于SeqScan和Vector Adapter之间还是一个火山模型的单行操作，这里理论上是可以做融合的

```sql
TestDB=# explain analyze select * from test group by c1;
                                                          QUERY PLAN                                                           
-------------------------------------------------------------------------------------------------------------------------------
 Row Adapter  (cost=42.02..42.02 rows=200 width=4) (actual time=0.450..0.450 rows=0 loops=1)
   ->  Vector Sonic Hash Aggregate  (cost=40.02..42.02 rows=200 width=4) (actual time=0.447..0.447 rows=0 loops=1)
         Group By Key: c1
         ->  Vector Adapter(type: BATCH MODE)  (cost=34.02..34.02 rows=2402 width=4) (actual time=0.010..0.010 rows=0 loops=1)
               ->  Seq Scan on test  (cost=0.00..34.02 rows=2402 width=4) (actual time=0.002..0.002 rows=0 loops=1)
 Total runtime: 2.744 ms
(6 rows)
```

2. openGauss的向量化引擎社区更新比较少，可能实现的并不是那么完善
3. 列式的内存batch结构可能对于TP数据库来说并不是最优解，因为TP数据库操作都是以行为单位，即大多为增删改查一整行记录，所以显然把一行数据存在物理上相邻的位置是个很好的选择。例如插入一行数据，行存一次IO就可以了，但对列存来说，就可能需要多次IO，update，delete也是类似的

**为什么不参考开源pg的批处理模型？**

  之前有人给pg实现过一个批处理模型，https://github.com/zhangh43/vectorize_engine 

  这个模型和向量化的区别就是它执行器处理的内存结构batch不是列式存储的，而是行式存储，它的batch结构可以理解成是N个TupleTableSlot组成的数组，这可以达到批处理的效果，作者也给出了测试数据，在TPCH上有不错的性能提升，但这个实现有一个弊端就是这个batch结构也就是VectorTupleSlot是和pg原有的TupleTableSlot强绑定的，这可能会导致在内存使用上不会达到最优的效果，而且TupleTableSlot这个结构历史包袱太重了，在设计上很复杂

## 所以要实现怎样一个执行器？

1. 实现一个内存紧密的行式batch结构，保证单个batch内的所有行的数据是连续存储的，针对单列的batch做特殊优化，单列标量类型数量利用SIMD提速，例如只针对某一列标量做agg或者hash
2. 需要考虑batch结构和work_mem之间的关系
3. 实现一套基于行式batch的存储层接口
4. 对象池复用batch来减少内存申请释放的开销
5. 并行化框架
6. pipeline异步化？？？push模型？？？
7. 

