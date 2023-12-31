MySQL INSERT 流程

一条 insert 语句在写入磁盘的过程中到底涉及了哪些文件？顺序又是如何的？下面我们用两张图和大家一 起解析 insert 语句的磁盘写入之旅。

图 1：事务提交前的日志文件写入

过程： 

1. 首先 insert 进入 server 层后，会进行一些必要的检查，检查的过程中并不会涉及到磁盘的写入。 
2. 检查没有问题之后，便进入引擎层开始正式的提交。我们知道 InnoDB 会将数据页缓存至内存中的 buffer pool，所以 insert 语句到了这里并不需要立刻将数据写入磁盘文件中，只需要修改 buffer pool 当中对应的数据页就可以了。  buffer pool 中的数据页刷盘并不需要在事务提交前完成，其中交互过程会在下一张图中分解。 
3.  但仅仅写入内存的 buffer pool 并不能保证数据的持久化，如果 MySQL 宕机重启了，需要保证 insert 的数据不会丢失。redo log 因此而生，当 innodb_flush_log_at_trx_commit=1 时，每次事务提交都会 触发一次 redo log 刷盘。（redo log 是顺序写入，相比直接修改数据文件，redo 的磁盘写入效率更加 高效） 
4.  如果开启了 binlog 日志，我们还需将事务逻辑数据写入 binlog 文件，且为了保证复制安全，建议使 用 sync_binlog=1 ，也就是每次事务提交时，都要将 binlog 日志的变更刷入磁盘。 综上（在 InnoDB buffer pool 足够大且上述的两个参数设置为双一时），insert 语句成功提交时，真正发生 磁盘数据写入的，并不是 MySQL 的数据文件，而是 redo log 和 binlog 文件。 然而，InnoDB buffer pool 不可能无限大，redo log 也需要定期轮换，很难容下所有的数据，下面我们就来 看看 buffer pool 与磁盘数据文件的交互方式。

```
double write 
​	InnoDB buffer pool 一页脏页大小为 16 KB，如果只写了前 4KB 时发生宕机，那这个脏页就发生了 写失败，会造成数据丢失。为了避免这一问题，InnoDB 使用了 double write 机制（InnoDB 将 double write 的数据存于共享表空间中）。在写入数据文件之前，先将脏页写入 double write 中，当然这里的写入 都是需要刷盘的。有人会问 redo log 不是也能恢复数据页吗？为什么还需要 double write？这是因为 redo log 中记录的是页的偏移量，比如在页偏移量为 800 的地方写入数据 xxx，而如果页本身已经发生损坏， 应用 redo log 也无济于事。 

insert buffer 
​	InnoDB 的数据是根据聚集索引排列的，通常业务在插入数据时是按照主键递增的，所以插入聚集索 引一般是顺序磁盘写入。但是不可能每张表都只有聚集索引，当存在非聚集索引时，对于非聚集索引的变 更就可能不是顺序的，会拖慢整体的插入性能。为了解决这一问题，InnoDB 使用了 insert buffer 机制，将 对于非聚集索引的变更先放入 insert buffer ，尽量合并一些数据页后再写入实际的非聚集索引中去。
```

![image-20230524100819133](C:\Users\itwb_lixl\AppData\Roaming\Typora\typora-user-images\image-20230524100819133.png)

![image-20230524100734811](C:\Users\itwb_lixl\AppData\Roaming\Typora\typora-user-images\image-20230524100734811.png)

图 2：事务提交后的数据文件写入 

过程： 

1. 当 buffer pool 中的数据页达到一定量的脏页或 InnoDB 的 IO 压力较小 时，都会触发脏页的刷盘操 作。
2. 当开启 double write 时，InnoDB 刷脏页时首先会复制一份刷入 double write，在这个过程中，由于 double write 的页是连续的，对磁盘的写入也是顺序操作，性能消耗不大。 
3. 无论是否经过 double write，脏页最终还是需要刷入表空间的数据文件。刷入完成后才能释放 buffer pool 当中的空间。 
4. insert buffer 也是 buffer pool 中的一部分，当 buffer pool 空间不足需要交换出部分脏页时，有可能将 insert buffer 的数据页换出，刷入共享表空间中的 insert buffer 数据文件中。 
5. 当 innodb_stats_persistent=ON 时，SQL 语句所涉及到的 InnoDB 统计信息也会被刷盘到 innodb_table_stats 和 innodb_index_stats 这两张系统表中，这样就不用每次再实时计算了。 
6. 有一些情况下可以不经过 double write 直接刷盘 a. 关闭 double write b. 不需要 double write 保障，如 drop table 等操作 汇总两张图，一条 insert 语句的所有涉及到的数据在磁盘上会依次写入 redo log，binlog，(double write， insert buffer) 共享表空间，最后在自己的用户表空间落定为安。