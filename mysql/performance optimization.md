#  performance optimization 

- **问题背景** 


有一个用户使用 DB 的时候遇到了问题，现有的 DB 性能无法满足用户的性能需求。
用户在对现有的 DB 进行压力测试时发现 QPS + TPS 小于 7W/S，
继续加大压力的时候 Load 上涨、Idle CPU 很低、Thread running 飙升、性能下降，
最终导致网站处理并发能力的下降，无法达到预期的吞吐量。用户在对现有逻辑及吞吐量计算的基础上提出了性能指标，
即 DB 的单机性能 QPS + TPS 大于 10W/S， 只有这样才能满足业务要求，否则 DB 就是整个链路的瓶颈。


- **现场信息收集**


- 首先，使用监控工具查看了用户实例的性能状态，如下所示： 


      -----load-avg---- ---cpu-usage--- ---------QPS---------------TPS------Hit%-----------threads------ 
      
      
      1m    5m   15m |usr sys idl iow|  ins   upd   del    sel   iud|     lor    hit| run  con  cre  cac|
   
   
      99.86 46.70 18.68| 65  25  10   0| 1045  3411     0  62800  4456|  304288 100.00| 251  963    0    0|
   
   
      99.86 46.70 18.68| 64  25  11   0| 1005  2956     0  64017  2961|  311912 100.00| 299  963    0    0|
   
   
      99.86 46.70 18.68| 66  25   9   0| 1223  3274     0  64511  4497|  309941 100.00| 325  963    0    0|
   
   
      99.86 46.70 18.68| 66  24  10   0| 1188  2992     0  64425  4180|  413496 100.00| 331  963    0    0|
   
   
      97.86 46.70 18.68| 66  24  10   0| 1148  3319     0  63536  4467|  307956 100.00| 262  963    0    0|
   
   
      97.86 46.70 18.68| 63  25  11   0| 1162  3723     0  64628  4485|  300747 100.00| 306  963    0    0|
   
   
      97.86 46.70 18.68| 66  25   9   0| 1117  4416     0  64077  5533|  305845 100.00| 273  963    0    0|
   
   
      96.86 46.70 18.68| 64  24  12   0| 1101  4240     0  63520  5341|  307361 100.00| 234  963    0    0|
   
   
      96.86 46.70 18.68| 65  25  10   0| 1128  4502     0  62604  5630|  312940 100.00|  52  963    0    0|
    
    
      96.86 46.92 18.68| 65  25  10   0|  900  4846     0  60648  5746|  298142 100.00| 282  963    0    0|
      
      
      
      
     从上面的性能信息可以发现命中率 100%， 即用户基本是全内存操作，thread running 较高，
     CPU 有少量， thread running 彪升，到底线程在做什么呢？怀着这样的疑问，然后执行了一
     下 pstack {pid of mysqld} > pid.info 以获取实例的内部线程信息，然后使用 pt-pmp pid.info 将
     堆栈信息进行显示，发现了如下的信息：
      
      
      
      192 __lll_lock_wait(libpthread.so.0),_L_lock_974(libpthread.so.0),pthread_mutex_lock(libpthread.so.0),inline_mysql_mutex_lock(mysql_thread.h:690),lock(mysql_thread.h:690),open_table(mysql_thread.h:690),open_and_process_table(sql_base.cc:4726),open_tables(sql_base.cc:4726),open_normal_and_derived_tables(sql_base.cc:5856),execute_sqlcom_select(sql_parse.cc:5129),mysql_execute_command(sql_parse.cc:2656),mysql_parse(sql_parse.cc:6408),dispatch_command(sql_parse.cc:1340),do_command(sql_parse.cc:1037),do_handle_one_connection(sql_connect.cc:990),handle_one_connection(sql_connect.cc:906),pfs_spawn_thread(pfs.cc:1860),start_thread(libpthread.so.0),clone(libc.so.6)
      31 __lll_lock_wait(libpthread.so.0),_L_lock_790(libpthread.so.0),pthread_mutex_lock(libpthread.so.0),rw_pr_wrlock(thr_rwlock.c:397),inline_mysql_prlock_wrlock(mysql_thread.h:984),MDL_map_partition::move_from_hash_to_lock_mutex(mysql_thread.h:984),find_or_insert(mdl.cc:898),MDL_map::find_or_insert(mdl.cc:898),try_acquire_lock_impl(mdl.cc:2033),MDL_context::acquire_lock(mdl.cc:2033),open_table_get_mdl_lock(sql_base.cc:2587),open_table(sql_base.cc:2587),open_and_process_table(sql_base.cc:4726),open_tables(sql_base.cc:4726),open_normal_and_derived_tables(sql_base.cc:5856),execute_sqlcom_select(sql_parse.cc:5129),mysql_execute_command(sql_parse.cc:2656),mysql_parse(sql_parse.cc:6408),dispatch_command(sql_parse.cc:1340),do_command(sql_parse.cc:1037),do_handle_one_connection(sql_connect.cc:990),handle_one_connection(sql_connect.cc:906),pfs_spawn_thread(pfs.cc:1860),start_thread(libpthread.so.0),clone(libc.so.6)
      ...
      ...
      1 lfind(libc.so.6),lsearch(lf_hash.c:267),lf_hash_search(lf_hash.c:267),find_or_create_digest(pfs_digest.cc:223),end_statement_v1(pfs.cc:4815),inline_mysql_end_statement(mysql_statement.h:215),dispatch_command(mysql_statement.h:215),do_command(sql_parse.cc:1037),do_handle_one_connection(sql_connect.cc:990),handle_one_connection(sql_connect.cc:906),pfs_spawn_thread(pfs.cc:1860),start_thread(libpthread.so.0),clone(libc.so.6)
    
    根据上述的 pt-pmp & pstack 文件相结合，可以看到如下堆栈： 
    


      #0  0x00007fc5f03eef7d in __lll_lock_wait () from /lib64/libpthread.so.0
      #1  0x00007fc5f03ead77 in _L_lock_974 () from /lib64/libpthread.so.0
      #2  0x00007fc5f03ead20 in pthread_mutex_lock () from /lib64/libpthread.so.0
      #3  0x00000000006a64e8 in inline_mysql_mutex_lock (src_line=115, src_file=0xc73308 "../sql/table_cache.h", that=0x138d9e0 <table_cache_manager>) at ../include/mysql/psi/mysql_thread.h:690
      #4  lock (this=0x138d9e0 <table_cache_manager>) at ../sql/table_cache.h:115
      #5  open_table (thd=thd@entry=0x397e5210, table_list=table_list@entry=0x7fbd44004e20, ot_ctx=ot_ctx@entry=0x7fbd740876e0) at ../sql/sql_base.cc:2944
      #6  0x00000000006aed49 in open_and_process_table (ot_ctx=0x7fbd740876e0, has_prelocking_list=false, prelocking_strategy=0x7fbd74087920, flags=0, counter=0x397e7050, tables=0x7fbd44004e20, lex=0x397e6fa0, thd=0x397e5210) at ../sql/sql_base.cc:4726
      #7  open_tables (thd=thd@entry=0x397e5210, start=start@entry=0x7fbd74087918, counter=0x397e7050, flags=flags@entry=0, prelocking_strategy=prelocking_strategy@entry=0x7fbd74087920) at ../sql/sql_base.cc:5159
      #8  0x00000000006af7fa in open_normal_and_derived_tables (thd=thd@entry=0x397e5210, tables=0x7fbd44004e20, flags=flags@entry=0) at ../sql/sql_base.cc:5856
      #9  0x0000000000572a48 in execute_sqlcom_select (thd=thd@entry=0x397e5210, all_tables=<optimized out>) at ../sql/sql_parse.cc:5129
      #10 0x00000000006fa010 in mysql_execute_command (thd=thd@entry=0x397e5210) at ../sql/sql_parse.cc:2656
    
    
    
    根据上面收集的信息我们可以清楚的得出以下结论：
    
    - 应用在执行SQL语句的过程中，table_cache_manager 中的锁冲突比较严重；
    
    
    - MySQL Server 层中的 MDL_lock 冲突比较重；
    
    
    - 实例开启了 Performance_schema 功能；
    
    
    经过了上面的分析，我们需要着重查看上述问题的相关变量，变量设置的情况会对性能造成直接的影响，执行结果如下：
    
    
``` 
MySQL [(none)]> show variables like '%performance_schema'; 
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| performance_schema | ON    |
+--------------------+-------+
1 row in set (0.00 sec)

MySQL [(none)]> show variables like '%instances';
+-----------------------------------------+--------+
| Variable_name                           | Value  |
+-----------------------------------------+--------+
| innodb_buffer_pool_instances            | 8      |
| metadata_locks_hash_instances           | 8      |
| ........................................|........|
| ........................................|........|
| table_open_cache_instances              | 1      |
+-----------------------------------------+--------+
10 rows in set (0.00 sec)
```  
 
-  **参数分析** 



这里我们先来介绍一下上述参数在 MySQL 中的作用 & 含义： 


```
table_open_cache_instances 简介
table_open_cache_instances 指的是 MySQL 缓存 table 句柄的分区的个数，而每一个 cache_instance
可以包含不超过 table_open_cache/table_open_cache_instances 的table_cache_element，
```


MySQL 打开表的过程可以简单的概括为：



```
1. 根据线程的 thread_id 确定线程将要使用的 table_cache，即 thread_id % table_cache_instances;
2. 从该 tabel_cache 元素中查找相关系连的 table_cache_element，如果存在转 3，如果不存在转 4； 
3. 从 2 中查找的table_cache_element 的 free_tables 中出取一个并返回，并调整 table_cache_element 中的 free_tables & used_tables 元素；
4. 如果 2 中不存在，则重新创建一个 table, 并加入对应的 table_cache_element 的 used_tables的列表；
```

```
从以上过程可以看出，MySQL 在打开表的过程中会首先从 table_cache 中进行查找有没有可
以直接使用的表句柄，有则直接使用，没有则会创建并加入到对应的 table_cache 中对应的 
table_cache_element 中，从刚才提取的现场信息来看，有大量的线程在查找 table_cache 的
过程中阻塞着了，而 table_open_cache_instances 的个数为 1， 因此，此参数的设置需要调 
整，由于 table_open_cache_instances 的大小和 线程 ID & 并发 有关系，考虑当前的并发是
1000左右，于是将该植设置为 32；
```    
    
    
```
MySQL 中不同的线程虽然使用各自的 table 句柄，但是共享着同一个table_share，如果想从
源码上了解 table & table_share 以及 两者之间的相互，可以从变量 table_open_cache， 
table_open_cache_instances，table_definition_cache 入手，阅读 Table_cache_manager， 
Table_cache， Table_cache::get_table 等相关代码
```




 
-  **MDL Lock 的前世今生** 


MDL_LOCK 就是为了解决上述问题而在 5.5 中引入的。简单的说 MDL Lock 是 MySQL Server 层中的表锁，
主要是为了控制 Server 层 DDL & DML 的并发而设计的， 但是 5.5 的设计中只有一把大锁，所以到5.6中添加了参数
metadata_locks_hash_instances 来控制分区的数量，进而实现大锁的拆分，虽然锁的拆分提
高了并发的性能，但是仍然存在着不少的性能问题，所以在 5.7.4 中 MDL Lock 的实现方式采
用了 lock free 算法，彻底的解决了 Server 层表锁的性能问题，而参数 
metadata_locks_hash_instances 也将会在之后的某个版本中被删除掉


由于实例中的表的数目比较多，而 metadata_locks_hash_instances 的参数设置仅为8，因
此，为了将底锁的冲突的可能性，我们将此值设置为 32；


-  ** Performance Schema 作用 & 影响


通俗的说，performance schema 是 MySQL 的内部诊断器，用于记录 MySQL 在运行期间的
各种信息，如表锁情况、mutex 竟争情况、执行语句的情况等，和 Information Schema 类似
的是拥用的信息都是内存信息，而不是存在磁盘上的，但和 information_schema 有以下不同点：



- information_schema 中的信息都是从 MySQL 的内存对象中读出来的，只有在需要的时候 
  才会读取这些信息，如 processlist, profile, innodb_trx 等，不需要额外的存储或者函数调
  用，而 performance schema 则是通过一系列的回调函数来将这些信息额外的存储起来，
  使用的时候进行显示，因此 performance schema 消耗更多的 CPU & memory 资源； 
  
 -  Information_schema 中的表是没有表定义文件的，表结构是在内存中写死的，而 
    performation_schema 中的表是有表定义文件的； 
  
 -  实现方式不同，Information_schema 只是对内存的 traverse copy, 而  
    performance_schema 则使用固定的接口来进行实现；
    
 -  作用不同，Information_schema 主要是轻量级的使用，即使在使用的时候也很少会有性能 
    影响，performance_schema 则是 MySQL 的内部监控系统，可以很好的定位性能问题，
    但也很影响性能； 
    
    
    
   由以上的分析不难看出，在性能要求比较高的情况下，关闭 performance_schema 是一个不错 
   的选择，因此将 performance_schema 关闭。另外关闭 performance_schema 的一个原因则
   是因为它本身的稳定性，因为之前在使用 performance_schema 的过程中遇到了不稳定的问
   题，当然，遇到一个问题我们就会修复一个，只是考虑到性能问题，我们暂时将其关闭。 
   
   
   经过上面的分析和判断，我们对参数做了如下的调整：  
   
   
   ```
   table_open_cache_instances=32
   metadata_locks_hash_instances=32
   performance_schema=OFF
   innodb_purge_threads=4
   
   ``` 

 - **勉强解决问题** 
 
 
 调整了以上参数后，我们重启实例，然后要求客户做新一轮的压力测试，测试部分数据如下： 
 
 
 ```
 -----load-avg---- ---cpu-usage--- ---------QPS---------------TPS------Hit%-----------threads------ 
  1m    5m   15m |usr sys idl iow|  ins   upd   del    sel   iud|     lor     hit| run  con  cre  cac|
 9.91  2.75  3.16| 24  14  62   0|    0  3698     0   87124  3698|  304288 100.00|  23  963    0    0|
 9.91  2.75  3.16| 41  24  35   0|    0  7191     0  124506  7191|  311912 100.00|  20  963    0    0|
12.48  3.41  3.37| 45  24  31   0|  352  8547     0  122248  8899|  309941 100.00|  35  963    0    0|
12.48  3.41  3.37| 55  27  18   0| 1514  7338     0  118130  8852|  413496 100.00| 217  963    0    0|
12.48  3.41  3.49| 60  27  13   0| 1815  6778     0  108114  8593|  307956 100.00|  20  963    0    0|
12.48  3.41  3.49| 58  25  17   0| 1909  7575     0  102566  9484|  300747 100.00|  17  963    0    0|
12.48  3.78  3.49| 57  23  20   0| 2022  7893     0  101197  9915|  305845 100.00|  21  963    0    0|
13.86  3.78  3.68| 58  26  16   0| 1732  7498     0  104869  9230|  307361 100.00|  27  963    0    0|
13.86  4.38  3.68| 57  24  19   0| 2067  6093     0  106261  8160|  312940 100.00|  33  963    0    0|
15.86  4.38  3.68| 57  24  19   0| 2008  5661     0  102623  7669|  298142 100.00|  12  963    0    0|

```


从以上的测试数据来看， QPS + TPS > 10W 已经满足要求，通过 perf top -p {pidof mysqld} 
命令查看了一下系统负载，发现了一处比较吃 CPU 的地方 ut_delay，详情如下： 

```
 20.69%  mysqld               [.] ut_delay(unsigned long)
  4.86%  mysqld               [.] my_hash_sort_utf8
  3.94%  mysqld               [.] mutex_spin_wait(ib_mutex_t*, char const*, unsigned long)
  3.59%  mysqld               [.] read_view_open_now_low(unsigned long, mem_block_info_t*)
  3.13%  [kernel]             [k] _raw_spin_lock
  3.05%  mysqld               [.] my_ismbchar_utf8
  2.77%  mysqld               [.] my_charpos_mb
  2.29%  mysqld               [.] my_strnxfrm_unicode
  1.80%  mysqld               [.] MYSQLparse(THD*)
  1.61%  libc-2.17.so         [.] __memcpy_ssse3_back
  1.58%  libpthread-2.17.so   [.] pthread_mutex_lock
  0.73%  mysqld               [.] row_search_for_mysql(unsigned char*, unsigned long, row_prebuilt_t*, unsigned long, unsigned long)
  0.73%  mysqld               [.] my_convert
  0.71%  libc-2.17.so         [.] _int_malloc

```



使用 perf record & perf report 进行分析，发现调用比较多的地方是： mutex_spin_wait，
于是断定 Innodb 底层资源冲突比较严重，根据以往的经验执行如下命令：


```
mysql> show variables like '%spin%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| innodb_spin_wait_delay | 10000 |
| innodb_sync_spin_loops | 30    |
+------------------------+-------+
2 rows in set (5.55 sec)

``` 


在 MySQL 内部，当 innodb 线程获取 mutex 资源而得不到满足时，会最多进行 
innodb_sync_spin_loops 次尝试获取 mutex 资源，每次失败后会调用 
ut_delay(ut_rnd_interval(0, srv_spin_wait_delay)，导致 ut_delay 占用了过多的 CPU， 其中  
ut_delay 的定义如下：

```

/*************************************************************//**
Runs an idle loop on CPU. The argument gives the desired delay
in microseconds on 100 MHz Pentium + Visual C++.
@return dummy value */
UNIV_INTERN
ulint
ut_delay(
/*=====*/
        ulint   delay)  /*!< in: delay in microseconds on 100 MHz Pentium */
{
        ulint   i, j;
        j = 0;
        for (i = 0; i < delay * 50; i++) {
                j += i;
                UT_RELAX_CPU();
        }
        if (ut_always_false) {
                ut_always_false = (ibool) j;
        }

        return(j);
}


```


由于这两个值的设定取决于实例的负载以及资源的竟争情况，所以不断的尝试设置这两个参数
的值，经过多次的尝试最终将这两个参数分别设置为：innodb_spin_wait_delay = 6， 
innodb_sync_spin_loops = 20 (请注意这两个值不是推荐值！！！) 才将 ut_delay 的占用资源
降下来，最终降低了不必要的 CPU 消耗的同时 idle cpu 也稳定在了 20+，具体资源占用详情
如下： 

  ```
   
  6.52%  mysqld               [.] my_hash_sort_utf8
  3.93%  [kernel]             [k] _raw_spin_lock
  3.82%  mysqld               [.] my_ismbchar_utf8
  3.65%  mysqld               [.] my_charpos_mb
  3.31%  mysqld               [.] my_strnxfrm_unicode
  3.09%  mysqld               [.] ut_delay(unsigned long)
  2.58%  mysqld               [.] read_view_open_now_low(unsigned long, mem_block_info_t*)
  2.38%  mysqld               [.] MYSQLparse(THD*)
  1.91%  libc-2.17.so         [.] __memcpy_ssse3_back
  1.89%  mysqld               [.] mutex_spin_wait(ib_mutex_t*, char const*, unsigned long)
  1.79%  libpthread-2.17.so   [.] pthread_mutex_lock
  
  ```
  
  
  优化到这个地步似乎达到了客户要求的性能，即 DB 单机性能为 QPS + TPS > 10W，可是如果并发量在加大，我们的 DB 能扛住更高的压力吗？
  
  
  
  - **又起波澜**  
  
  经过上面参数的调整，DB 已经不是性能的瓶颈，应用的吞吐量由之前的 1100 -> 1400+，但 
  是离 2000 的吞吐量还比较远，瓶颈出现在了应用端，为了增加吞吐量，客户又增加了几台客
  户端机器，连接数也由之前的 900+ 上升到 1000+，此时发现 DB 虽然能够响应，但偶尔会出
  现 thread running 飙高的情况，具体运行状态如下，其中 mysql_com_tps = 
  (mysql_com_insert + mysql_com_update + mysql_com_delete)： 
  
  
  
  ```
  
  
  +---------------------+--------+---------+------------------+------------------+------------------+------------------+---------------+-----------------+---------------------+
| dtime               | load_1 | idlecpu | mysql_com_insert | mysql_com_update | mysql_com_delete | mysql_com_select | mysql_com_tps | threads_running | threads_connections |
+---------------------+--------+---------+------------------+------------------+------------------+------------------+---------------+-----------------+---------------------+
| 2016-07-03 21:48:24 |   5.78 |   30.00 |             1706 |             6573 |                0 |            97359 |          8279 |              25 |                1011 |
| 2016-07-03 21:48:25 |   5.78 |   41.00 |             1606 |             7018 |                0 |            98273 |          8624 |              17 |                1011 |
| 2016-07-03 21:48:26 |   5.78 |   33.00 |             1616 |             5739 |                0 |            93752 |          7355 |             108 |                1011 |
| 2016-07-03 21:48:27 |   5.78 |   39.00 |             1505 |             6436 |                0 |            91381 |          7941 |              12 |                1011 |
| 2016-07-03 21:48:28 |   8.20 |   36.00 |             1849 |             4514 |                0 |            87881 |          6363 |             156 |                1011 |
| 2016-07-03 21:48:29 |   8.20 |   28.00 |             1702 |             6386 |                0 |            97621 |          8088 |              35 |                1011 |
| 2016-07-03 21:48:30 |   8.20 |   42.00 |             1442 |             6708 |                0 |            94920 |          8150 |              24 |                1011 |
| 2016-07-03 21:48:31 |   8.20 |   28.00 |             1399 |             8283 |                0 |            98801 |          9682 |             189 |                1011 |
| 2016-07-03 21:48:32 |   8.20 |   28.00 |             1254 |             7960 |                0 |            90461 |          9214 |             137 |                1011 |
| 2016-07-03 21:48:33 |   9.86 |   23.00 |             1039 |             7557 |                0 |            92145 |          8596 |             193 |                1011 |
| 2016-07-03 21:48:34 |   9.86 |   36.00 |             1358 |             7696 |                0 |            85274 |          9054 |             301 |                1011 |
| 2016-07-03 21:48:35 |   9.86 |   39.00 |             1069 |             8148 |                0 |            80185 |          9217 |             346 |                1011 |
| 2016-07-03 21:48:36 |   9.86 |   44.00 |             1019 |             8484 |                0 |            77787 |          9503 |             378 |                1011 |
| 2016-07-03 21:48:37 |   9.86 |   41.00 |             1023 |             7290 |                0 |            74965 |          8313 |             341 |                1011 |
| 2016-07-03 21:48:38 |  10.36 |   39.00 |              987 |             8031 |                0 |            83857 |          9018 |             279 |                1011 |
| 2016-07-03 21:48:39 |  10.36 |   39.00 |             1108 |             7165 |                0 |            84070 |          8273 |             255 |                1011 |
| 2016-07-03 21:48:40 |  10.36 |   42.00 |             1219 |             5804 |                0 |            80959 |          7023 |              22 |                1011 |
| 2016-07-03 21:48:41 |  10.36 |   40.00 |             1034 |             6546 |                0 |            82380 |          7580 |             296 |                1011 |
| 2016-07-03 21:48:42 |  10.36 |   41.00 |              809 |             5973 |                0 |            79554 |          6782 |             319 |                1011 |
| 2016-07-03 21:48:43 |  10.65 |   39.00 |              949 |             7252 |                0 |            79690 |          8201 |             312 |                1011 |
  
 
 ```
 
 
-  **查找问题原因**


Thread running 的偶尔飙升引起了我的注意，说明内部必然有冲突，随着压力和并发量的不断
增大，应用可能会受到类似之前的影响，因此很有必要查看其中的原因并尽最大的努力解决
之。通过仔细的观察 thread running & mysql com 信息，当 thread running 较高 & com 信息
较低的时候，执行了 pt-pmp -p {pid of mysqld}，抓到了以下信息：

 
```

179 pthread_cond_wait,os_cond_wait(os0sync.cc:214),os_event_wait_low(os0sync.cc:214),sync_array_wait_event(sync0arr.cc:424),mutex_spin_wait(sync0sync.cc:580),mutex_enter_func(sync0sync.ic:218),pfs_mutex_enter_func(sync0sync.ic:218),read_view_remove(sync0sync.ic:218),read_view_close_for_mysql(sync0sync.ic:218),ha_innobase::external_lock(ha_innodb.cc:12326),handler::ha_external_lock(handler.cc:7190),unlock_external(lock.cc:646),mysql_unlock_tables(lock.cc:389),mysql_unlock_some_tables(lock.cc:389),JOIN::optimize(sql_optimizer.cc:406),mysql_execute_select(sql_select.cc:1087),mysql_select(sql_select.cc:1087),handle_select(sql_select.cc:110),execute_sqlcom_select(sql_parse.cc:5156),mysql_execute_command(sql_parse.cc:2656),mysql_parse(sql_parse.cc:6408),dispatch_command(sql_parse.cc:1340),do_command(sql_parse.cc:1037),do_handle_one_connection(sql_connect.cc:990),handle_one_connection(sql_connect.cc:906),pfs_spawn_thread(pfs.cc:1860),start_thread(libpthread.so.0),clone(libc.so.6)
    150 pthread_cond_wait,os_cond_wait(os0sync.cc:214),os_event_wait_low(os0sync.cc:214),sync_array_wait_event(sync0arr.cc:424),mutex_spin_wait(sync0sync.cc:580),mutex_enter_func(sync0sync.ic:218),pfs_mutex_enter_func(sync0sync.ic:218),read_view_open_now(sync0sync.ic:218),trx_assign_read_view(trx0trx.cc:1481),row_search_for_mysql(row0sel.cc:4090),ha_innobase::index_read(ha_innodb.cc:7516),handler::index_read_idx_map(handler.cc:6846),handler::ha_index_read_idx_map(handler.cc:2787),join_read_(handler.cc:2787),join_read__table(handler.cc:2787),make_join_statistics(sql_optimizer.cc:3592),JOIN::optimize(sql_optimizer.cc:363),mysql_execute_select(sql_select.cc:1087),mysql_select(sql_select.cc:1087),handle_select(sql_select.cc:110),execute_sqlcom_select(sql_parse.cc:5156),mysql_execute_command(sql_parse.cc:2656),mysql_parse(sql_parse.cc:6408),dispatch_command(sql_parse.cc:1340),do_command(sql_parse.cc:1037),do_handle_one_connection(sql_connect.cc:990),handle_one_connection(sql_connect.cc:906),pfs_spawn_thread(pfs.cc:1860),start_thread(libpthread.so.0),clone(libc.so.6)
      7 __lll_lock_wait(libpthread.so.0),_L_cond_lock_973(libpthread.so.0),__pthread_mutex_cond_lock(libpthread.so.0),pthread_cond_wait,os_cond_wait(os0sync.cc:214),os_event_wait_low(os0sync.cc:214),sync_array_wait_event(sync0arr.cc:424),mutex_spin_wait(sync0sync.cc:580),mutex_enter_func(sync0sync.ic:218),pfs_mutex_enter_func(sync0sync.ic:218),read_view_open_now(sync0sync.ic:218),trx_assign_read_view(trx0trx.cc:1481),row_search_for_mysql(row0sel.cc:4090),ha_innobase::index_read(ha_innodb.cc:7516),handler::index_read_idx_map(handler.cc:6846),handler::ha_index_read_idx_map(handler.cc:2787),join_read_(handler.cc:2787),join_read__table(handler.cc:2787),make_join_statistics(sql_optimizer.cc:3592),JOIN::optimize(sql_optimizer.cc:363),mysql_execute_select(sql_select.cc:1087),mysql_select(sql_select.cc:1087),handle_select(sql_select.cc:110),execute_sqlcom_select(sql_parse.cc:5156),mysql_execute_command(sql_parse.cc:2656),mysql_parse(sql_parse.cc:6408),dispatch_command(sql_parse.cc:1340),do_command(sql_parse.cc:1037),do_handle_one_connection(sql_connect.cc:990),handle_one_connection(sql_connect.cc:906),pfs_spawn_thread(pfs.cc:1860),start_thread(libpthread.so.0),clone(libc.so.6)
      1 operator(read0read.cc:296),ut_list_map<ut_list_base<trx_t>,(read0read.cc:296),read_view_open_now_low(read0read.cc:296),read_view_open_now(read0read.cc:388),trx_assign_read_view(trx0trx.cc:1481),row_search_for_mysql(row0sel.cc:4090),ha_innobase::index_read(ha_innodb.cc:7516),handler::index_read_idx_map(handler.cc:6846),handler::ha_index_read_idx_map(handler.cc:2787),join_read_(handler.cc:2787),join_read__table(handler.cc:2787),make_join_statistics(sql_optimizer.cc:3592),JOIN::optimize(sql_optimizer.cc:363),mysql_execute_select(sql_select.cc:1087),mysql_select(sql_select.cc:1087),handle_select(sql_select.cc:110),execute_sqlcom_select(sql_parse.cc:5156),mysql_execute_command(sql_parse.cc:2656),mysql_parse(sql_parse.cc:6408),dispatch_command(sql_parse.cc:1340),do_command(sql_parse.cc:1037),do_handle_one_connection(sql_connect.cc:990),handle_one_connection(sql_connect.cc:906),pfs_spawn_thread(pfs.cc:1860),start_thread(libpthread.so.0),clone(libc.so.6)
      ...
      ...
 ```
 
 
 从上面的现场信息不难看出有很大一部分线程是在执行 read_view 的相关操作中被阻塞着了，
 那么什么是 read view，它的作用是什么，为什么会有大量的线程执行这个操作的时候被阻塞呢？
 
 
 **什么是 read view** 
 
 
 read view 又称读视图，用于存储事务创建时的活跃事务集合。当事务创建时，线程会对 
 trx_sys 上全局锁，然后遍历当前活跃事务列表，将当前活跃事务的ID存储在数组中的同时，
 记录最大事务 low_limit_id &　最小事务 high_limit_id & 最小序列化事务 low_limit_no。
 
 
 
 ** read view 的作用是什么 ** 
 
 
 Innodb record 格式包含 {记录头，主建，Trx_id，roll_ptr, extra_column} 等信息。 
 当事务执行时，凡是大于low_limit_id 的数据对于事务是不可见的，凡是事务小于 
 high_limit_id 的数据都是可见的，事务 ID 是 read_view 数组中的某一个时也是不可见的，
 Purge thread 在执行 Purge 操作时，凡是小于 low_limit_no 的数据，都是可以被 Purge 的，
 因此， read view 是 MySQL MVCC 实现的基础； 
 
 
 **为什么会有大量的线程阻塞** 
 
 事务创建时的步骤如下： 
 
 
 - 对 trx_sys->mutex 全局上锁； 
 
 - 顺序扫描 trx_sys->rw_trx_list，对 read_view 中的元素分配内存并进行赋值，主要包括活 
   跃事务ID的集合的创建，low_limit_id , high_limit_id, low_limit_no 等；
   
  - 将该 read_view 添加到有序列表 trx_sys->view_list中； 
  
  - 释放 trx_sys->mutex 锁； 
  
  
  由于read_view 的创建和销毁都需要获取 trx_sys->mutex, 当并发量很大的时候，事务链表会 
  比较长，又由于遍历本身也是一个费时的工作，所以此处便成为了瓶颈，既然我们遇到了这个问题，那么社区应该也有类似的问题。
  
  
  **read view 问题解决过程** 
  
 -  整个创建过程一直持有 trx_sys->mutex 锁；
 - read_view 的内存在每次创建中被分配，事务提交后被释放；
 - 需要遍历 trx_sys->trx_list (5.5) 或 trx_sys->rw_list (5.6)；
 - 并发较大，活跃事务链表过长时，会在 trx_sys->mutex 上有较大的消耗； 
 
 
 ** Percona read view 问题改进 ** 
 
 
 Percona 为了解决上述描述的问题，对trx_sys做了以下修改： 
 
 
 - 在 trx_sys下维护一个全局的事务ID的有序集合，事务的 创建 & 销毁 的同时将事务的 ID 从这个集合中移除；
 - 在 trx_sys下维护一个有序的已分配序列号的事务列表，已记录拥有最小序列号的事务，供 purge 时使用；
 - 减少不必要的内存分配，为每一个 trx_t 缓存一个 read_view，read_view 数组的大小根据创建时的活跃全局事务 ID 集合做必要的调整；
 
 
 做了上面的调整后，事务在创建过程中则不需要遍历 trx_sys->trx_list（version 5.5），
 直接使用 memcpy 即可获得活跃事务的ID，并且缓存的使用也大大减少了内存的不必要分配；
 
 
 
 - **线上效果** 
 
 
 鉴于当前存在的问题，为了解决客户的燃眉之急，决定上一个新版本，和客户联系后，可以重
 启实例，然后进行了替换操作，替换后的性能效果如下，可以看到 cpu 使用率、load、thread 
 running 降低的同时 QPS + TPS 性能上升，至此问题真正觉得问题应该解决了，余下的就是
 等客户的反馈了。
 
 ```
 
 +---------------------+--------+---------+------------------+------------------+------------------+------------------+---------------+-----------------+---------------------+
| dtime               | load_1 | idlecpu | mysql_com_insert | mysql_com_update | mysql_com_delete | mysql_com_select | mysql_com_tps | threads_running | threads_connections |
+---------------------+--------+---------+------------------+------------------+------------------+------------------+---------------+-----------------+---------------------+
| 2016-07-03 22:21:31 |   1.54 |   37.00 |             1995 |             8194 |                0 |           125782 |         10189 |              18 |                1012 |
| 2016-07-03 22:21:32 |   1.54 |   37.00 |             2205 |             8016 |                0 |           125974 |         10221 |              17 |                1012 |
| 2016-07-03 22:21:33 |   1.54 |   49.00 |             2061 |             5758 |                0 |           106469 |          7819 |              25 |                1012 |
| 2016-07-03 22:21:34 |   1.54 |   38.00 |             2450 |             7565 |                0 |           127511 |         10015 |              18 |                1012 |
| 2016-07-03 22:21:35 |   3.66 |   39.00 |             2121 |             6644 |                0 |           128277 |          8765 |              27 |                1012 |
| 2016-07-03 22:21:36 |   3.66 |   41.00 |             2617 |             5966 |                0 |           127987 |          8583 |              22 |                1012 |
| 2016-07-03 22:21:37 |   3.66 |   43.00 |             2009 |             6564 |                0 |           124135 |          8573 |              16 |                1012 |
| 2016-07-03 22:21:38 |   3.66 |   43.00 |             2294 |             5783 |                0 |           123519 |          8077 |              15 |                1012 |
| 2016-07-03 22:21:39 |   4.65 |   45.00 |             2050 |             6931 |                0 |           123719 |          8981 |              13 |                1012 |
| 2016-07-03 22:21:40 |   4.65 |   51.00 |             2039 |             5028 |                0 |           107993 |          7067 |              14 |                1012 |
| 2016-07-03 22:21:41 |   4.65 |   49.00 |             2041 |             5153 |                0 |           110077 |          7194 |              23 |                1012 |
| 2016-07-03 22:21:42 |   4.65 |   49.00 |             2215 |             5347 |                0 |           108539 |          7562 |              24 |                1012 |
| 2016-07-03 22:21:43 |   4.65 |   40.00 |             2000 |             7564 |                0 |           128957 |          9564 |              21 |                1012 |
| 2016-07-03 22:21:45 |   6.84 |   40.00 |             2218 |             6579 |                0 |           129526 |          8797 |              17 |                1012 |
| 2016-07-03 22:21:46 |   6.84 |   49.00 |             1859 |             5800 |                0 |           111708 |          7659 |              18 |                1012 |
| 2016-07-03 22:21:47 |   6.84 |   40.00 |             2098 |             6149 |                0 |           129089 |          8247 |              14 |                1012 |
| 2016-07-03 22:21:48 |   6.84 |   50.00 |             1761 |             4910 |                0 |           112003 |          6671 |              24 |                1012 |
| 2016-07-03 22:21:49 |   6.84 |   40.00 |             2283 |             7195 |                0 |           129313 |          9478 |              20 |                1012 |
| 2016-07-03 22:21:50 |   6.85 |   39.00 |             2394 |             6057 |                0 |           128982 |          8451 |              11 |                1012 |
| 2016-07-03 22:21:51 |   6.85 |   40.00 |             2160 |             5727 |                0 |           128516 |          7887 |              17 |                1012 |
| 2016-07-03 22:21:52 |   6.85 |   41.00 |             2405 |             5628 |                0 |           129897 |          8033 |              16 |                1012 |
| 2016-07-03 22:21:53 |   6.85 |   38.00 |             2072 |             7064 |                0 |           129313 |          9136 |              21 |                1012 |
 
 ```
 
 
 将监控数据入库，查看峰值 & 当时的负载情况，详情如下：
 
 
 ```
 
 MySQL [sysbench]> select dtime, (mysql_com_tps + mysql_com_select) as total_requests, mysql_com_select, mysql_com_tps, innodb_rows_update, idlecpu,threads_running from ip_10_108_107_97 order by total_requests desc;
+---------------------+----------------+------------------+---------------+--------------------+---------+-----------------+
| dtime               | total_requests | mysql_com_select | mysql_com_tps | innodb_rows_update | idlecpu | threads_running |
+---------------------+----------------+------------------+---------------+--------------------+---------+-----------------+
| 2016-07-03 22:15:17 |         170396 |           138029 |         32367 |              32357 |   44.00 |              19 |
| 2016-07-03 22:16:34 |         165842 |           136566 |         29276 |              28919 |   41.00 |              17 |
| 2016-07-03 22:15:14 |         164890 |           135873 |         29017 |              29007 |   43.00 |              16 |
| 2016-07-03 22:15:13 |         163007 |           131839 |         31168 |              31172 |   48.00 |              17 |
| 2016-07-03 22:15:16 |         162658 |           135986 |         26672 |              26687 |   44.00 |              13 |
| 2016-07-03 22:16:37 |         159783 |           134070 |         25713 |              25654 |   45.00 |              12 |
| 2016-07-03 22:16:35 |         156849 |           131101 |         25748 |              25609 |   44.00 |              23 |
| 2016-07-03 22:22:40 |         150488 |           129765 |         20723 |              19930 |   42.00 |              17 |
| 2016-07-03 22:22:38 |         148860 |           130581 |         18279 |              16767 |   39.00 |              25 |
| 2016-07-03 22:22:39 |         148797 |           131086 |         17711 |              16386 |   40.00 |              22 |
| 2016-07-03 22:16:36 |         148189 |           128710 |         19479 |              19418 |   44.00 |              21 |
| 2016-07-03 22:22:36 |         146731 |           129747 |         16984 |              14649 |   38.00 |              21 |
| 2016-07-03 22:22:37 |         145356 |           129450 |         15906 |              13810 |   39.00 |              24 |
| 2016-07-03 21:20:12 |         145331 |           128423 |         16908 |              15068 |   44.00 |              19 |
| 2016-07-03 21:20:10 |         145252 |           131791 |         13461 |              10909 |   43.00 |              10 |
| 2016-07-03 21:20:14 |         145200 |           127102 |         18098 |              17107 |   48.00 |              18 |
| 2016-07-03 21:19:09 |         144442 |           135123 |          9319 |               5555 |   42.00 |              19 |
| 2016-07-03 22:22:34 |         144181 |           129076 |         15105 |              12345 |   39.00 |              16 |
| 2016-07-03 22:22:33 |         143299 |           129232 |         14067 |              11302 |   39.00 |              15 |
| 2016-07-03 22:23:44 |         142831 |           130274 |         12557 |              10118 |   42.00 |              21 |
| 2016-07-03 21:19:10 |         142181 |           131958 |         10223 |               6051 |   43.00 |              18 |


 
 
 
 真的完美了吗，其实不是这样的，我们还有很多的事情要做，因为在解决问题的过程中，我们
 通过 pstack & pt-pmp 抓到了很多有用的信息，有一些是暂时没有解决的，如：
 
 
- innodb 内部表锁冲突严重；

- MDL Lock 即使扩大也存在着不小的影响；

- 内存分配也有一些需要优化的地方；

- 执行计划的计算代价比较高； 

- thread running 彪高时没有可以控制的方法； 

..........................
 
 
 
 
