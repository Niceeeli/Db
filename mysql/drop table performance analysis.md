
# analysis 
 
- **问题背景**


在删除大表的时候发现数据库性能有明显下降，并发的简单SELECT语句(查询的并不是正在删除的表)都会耗费数十秒才能返回结果；
 
 
 - **问题现象**
 
 
 在删除很多表，DROP TABLE语句在SHOW PROCESSLIST中显示状态为checking permissions，DML语句显示状态为Opening tables； 
 
 
 - **线上环境**
 
 
 线上的大表为InnoDB存储引擎，单个大表的ibd文件大小在50-80GB；实例的一些关键参数配置为：
* innodb_file_per_table=1
* innodb_buffer_pool_size=425GB
* innodb_buffer_pool_instances=8
可以看出，表对应的文件很大，buffer pool配置很高；

 
 - **复现步骤**

并发地DROP多个大表，同时监控简单SELECT语句的性能，并抓取mysqld的堆栈；


select count(*) from db1.t1;


count（*）


47648784



drop table db1.t1 


22.24 sec

select count(*) from db1.t2;


count（*）


39949893


drop table db1.t2;


28.21 sec


create table db1.t3(id int);


0s

insert into db1.t3 values (1);


0.00s 

select * from db1.t3；


id 


1


20.85s 

- **问题的原因** 
DROP表耗时的操作主要有两个：
* 当innodb_file_per_table为1时，删除一个表需要将buffer pool中属于这个tablespace的所有page无效；这个过程会持有
 buffer_pool mutex，和flush list mutex，当buffer pool配置很大时，会比较耗时；
 
* ext3删除文件系统上的ibd文件，当文件很大时，会比较耗时；


这两个耗时操作的实现是：在函数row_drop_table_for_mysql中，
先获取dict_sys->mutex，然后清理buffer pool(过程中会持有
buffer pool mutex)，再删除文件，最后释放dict_sys->mutex。
所以可以看出，当DROP一个大表时，会较长时间地排他性持有
dict_sys->mutex和buffer pool mutex


当这两个mutex被DROP TABLE长时间持有后，后续的SELECT
语句要么会在open_table的时候阻塞在dict_sys->mutex上，要么
会在读取表数据时(如果有可用的table cache)阻塞在buffer pool 
mutex上；对于后续的DROP TABLE，则会在删除table cache时
阻塞在dict_sys->mutex上，更糟糕的是，这些后续的DROP 
TABLE阻塞的时候是持有了所有的table_cache_instance的锁，也
会导致后续SELECT阻塞在Table_cache->lock()上；
所以从线上场景来看，大量的并发DROP大表会滚雪球一样导致DB
性能下降，也就会出现大量SELECT语句状态显示为Opening 
tables，大量DROP TABLE语句状态显示为checking 
permissions(验证过DROP TABLE阻塞在dict_sys->mutex上状
态会显示为checking permissions)。

总结来说：根本原因在于删除idb文件和清理buffer pool会比较耗时，
而这两个操作期间会持有dict_sys->mutex和buffer pool mutex，
阻塞后续DDL和DML。




- **解决方法** 
目前的两种思路： 
* 降低清理buffer pool和删除idb文件的时间
   Percona的lazy drop在清理buffer pool上做了优化，声称比MySQL 5.6的lazy drop性能更好；
 * 分析dict_sys->mutex和buffer pool mutex是否可以优化
 
 

  










