


#  common sql


**查询非innodb引擎的表**


    SELECT
    table_schema,
    table_name,
    engine,
    sys.format_bytes(data_length) as data_size
    FROM
    tables
    WHERE
    engine <> 'InnoDB' AND table_schema NOT IN ('mysql' , 'performance_schema','information_schema'); 
    
    
 **找出row_format 为fixed格式的表**
    
    
 select table_schema,table_name,row_format,CREATE_OPTIONS from information_schema.tables where CREATE_OPTIONS like '%fix%' ;
 
 
 **find all key**
 
 
 select constraint_schema,table_name,constraint_name,constraint_type from 
 information_schema.table_constraints where table_schema not in ('information_schema', 'mysql', 'test');
 
 
 **找出 自增字段 不在符合索引第一位 或者不是一个单独的key  影响 count(*) 的 table**
 
 
        Select A.TABLE_SCHEMA,A.TABLE_NAME
        From  (select TABLE_SCHEMA,TABLE_NAME,COLUMN_NAME From information_schema.columns where EXTRA='auto_increment' ) A,
        (select TABLE_SCHEMA,TABLE_NAME,COLUMN_NAME,ORDINAL_POSITION  From information_schema.KEY_COLUMN_USAGE   ) B
        Where A.TABLE_SCHEMA=B.TABLE_SCHEMA and A.TABLE_NAME=B.TABLE_NAME and A.COLUMN_NAME=B.COLUMN_NAME 
        Group by A.TABLE_SCHEMA,A.TABLE_NAME Having min(ORDINAL_POSITION) > 1 ; 
 
 
 
 
**找出没有创建主键的表**


    select concat(t.table_schema,'.',t.TABLE_NAME) as tablename 
    From information_schema.tables t left join (Select CONSTRAINT_SCHEMA,table_name from information_schema.TABLE_CONSTRAINTS where CONSTRAINT_TYPE='PRIMARY KEY') p 
    on t.table_name=p.table_name and t.TABLE_SCHEMA=p.CONSTRAINT_SCHEMA  
    Where t.table_schema not in ('performance_schema','information_schema','mysql') and p.table_name is  null ;  

    
    
    SELECT * FROM
    information_schema.tables t
    LEFT JOIN
    information_schema.STATISTICS s ON t.table_schema = s.table_schema
    AND t.table_name = s.table_name
    AND s.index_name = 'PRIMARY'
    WHERE -- 过滤系统表，不统计视图
    t.table_schema NOT IN ('mysql' , 'performance_schema',
    'information_schema','sys')
    AND table_type = 'base table'   
    AND s.index_name IS NULL;



 
 **列出被堵塞者以及持堵塞者的线程id 以及事务id 及query** 
 
 
 
    SELECT     r.trx_id waiting_trx_id,     r.trx_mysql_thread_id waiting_thread,     r.trx_query  
    wating_query,     b.trx_id blocking_trx_id,     b.trx_mysql_thread_id blocking_thread,     b.trx_query 
    blocking_query FROM     information_schema.innodb_lock_waits w         INNER JOIN  
    information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id         INNER JOIN 
    information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id; 



    SELECT lw.requesting_trx_id AS request_XID, 
    trx.trx_mysql_thread_id as request_mysql_PID,
    trx.trx_query AS request_query, 
    lw.blocking_trx_id AS blocking_XID, 
    trx1.trx_mysql_thread_id as blocking_mysql_PID,
    trx1.trx_query AS blocking_query, lo.lock_index AS lock_index FROM 
    information_schema.innodb_lock_waits lw INNER JOIN 
    information_schema.innodb_locks lo 
    ON lw.requesting_trx_id = lo.lock_trx_id INNER JOIN 
    information_schema.innodb_locks lo1 
    ON lw.blocking_trx_id = lo1.lock_trx_id INNER JOIN 
    information_schema.innodb_trx trx 
    ON lo.lock_trx_id = trx.trx_id INNER JOIN 
    information_schema.innodb_trx trx1 
    ON lo1.lock_trx_id = trx1.trx_id;



**按时间顺序排列出线程信息** 


select *From information_schema.processlist order by time desc  
select ID,USER,TIME,substring_index(HOST,':',1) as HOST,DB,COMMAND,STATE,INFO from information_schema.processlist 


**查看哪些索引采用部分索引（前缀索引）**


SELECT TABLE_SCHEMA, TABLE_NAME, INDEX_NAME, 
SEQ_IN_INDEX, COLUMN_NAME, CARDINALITY, SUB_PART
FROM INFORMATION_SCHEMA.STATISTICS WHERE 
SUB_PART > 10 ORDER BY SUB_PART DESC;


 
 **查看哪些索引长度超过30字节，重点查CHAR/VARCHAR/TEXT/BLOB等类型**
 
 
    select c.table_schema as `db`, c.table_name as `tbl`, 
    c.COLUMN_NAME as `col`, c.DATA_TYPE as `col_type`, 
    c.CHARACTER_MAXIMUM_LENGTH as `col_len`, 
    c.CHARACTER_OCTET_LENGTH as `col_len_bytes`,  
    s.NON_UNIQUE as `isuniq`, s.INDEX_NAME, s.CARDINALITY, 
    s.SUB_PART, s.NULLABLE 
    from information_schema.COLUMNS c inner join information_schema.STATISTICS s 
    using(table_schema, table_name, COLUMN_NAME) where 
    c.table_schema not in ('mysql', 'sys', 'performance_schema', 'information_schema', 'test') and 
    c.DATA_TYPE in ('varchar', 'char', 'text', 'blob') and 
    ((CHARACTER_OCTET_LENGTH > 20 and SUB_PART is null) or 
    SUB_PART * CHARACTER_OCTET_LENGTH/CHARACTER_MAXIMUM_LENGTH >20);
 
 
 
 **查看未完成的事务列表**
 
 
    select b.host, b.user, b.db, b.time, b.COMMAND, 
    a.trx_id, a. trx_state from 
    information_schema.innodb_trx a left join 
    information_schema.PROCESSLIST b on a.trx_mysql_thread_id = b.id;
 
 
 
**查看表数据列元数据信息**



    select a.table_id, a.name, b.name, b.pos, b.mtype, b.prtype, b.len from 
    information_schema.INNODB_SYS_TABLES a left join 
    information_schema.INNODB_SYS_COLUMNS b 
    using(table_id) where a.name = 'pp/t1';


 
**查看InnoDB表碎片率**
 
 
 
    SELECT TABLE_SCHEMA as `db`, TABLE_NAME as `tbl`, 
    1-(TABLE_ROWS*AVG_ROW_LENGTH)/(DATA_LENGTH + INDEX_LENGTH + DATA_FREE) AS `fragment_pct` 
    FROM information_schema.TABLES WHERE 
    TABLE_SCHEMA = 'test' ORDER BY fragment_pct DESC;



 
**查某个表在innodb buffer pool中的new block、old block比例**



    select table_name, count(*), sum(NUMBER_RECORDS), 
    if(IS_OLD='YES', 'old', 'new') as old_block from
    information_schema.innodb_buffer_page where 
    table_name = '`test`.`t1`' group by old_block;
 
 


**查看缓冲脏页**
    
    
    
    SELECT 
    pool_id,
    lru_position,
    space,
    page_number,
    table_name,
    oldest_modification,
    newest_modification
    FROM
    information_schema.INNODB_BUFFER_PAGE_LRU
    WHERE
    oldest_modification <> 0
    AND oldest_modification <> newest_modification;


 





 
 **查看库各个表自增id的使用情况** 
 
        SELECT   t.TABLE_NAME,   c.COLUMN_NAME,   ts.AUTO_INCREMENT 
        FROM   INFORMATION_SCHEMA.TABLE_CONSTRAINTS AS t,   
        information_schema.TABLES AS ts,   
        information_schema.KEY_COLUMN_USAGE AS c 
        WHERE   t.TABLE_NAME = ts.TABLE_NAME   AND ts.TABLE_NAME  = c.TABLE_NAME   
        AND  t.TABLE_SCHEMA = 'event' AND t.table_schema=ts.table_schema 
        AND t.table_schema=c.table_schema AND t.CONSTRAINT_TYPE = 'PRIMARY KEY'   ORDER BY ts.`AUTO_INCREMENT` DESC
 
 
 
 
 
  **查看ddl操作的未提交事务的线程id**
         
     select concat('kill ',i.trx_mysql_thread_id,';') from information_schema.innodb_trx i,
    (select  id, time  from information_schema.processlist  where  time = (select  max(time)  
    from  information_schema.processlist where  state = 'Waiting for table metadata lock'  and substring(info, 1, 5) 
    in ('alter' , 'optim', 'repai', 'lock ', 'drop ', 'creat'))) p  
    where timestampdiff(second, i.trx_started, now()) > p.time  and 
    i.trx_mysql_thread_id not in (connection_id(),p.id);
    
   **查看自增值的使用情况**
    
    SELECT table_schema, table_name, column_name, auto_increment,
    pow(2, case data_type
      when 'tinyint'   then 7
      when 'smallint'  then 15
      when 'mediumint' then 23
      when 'int'       then 31
      when 'bigint'    then 63
      end+(column_type like '% unsigned'))-1 as max_int
    FROM information_schema.tables t
    JOIN information_schema.columns c USING (table_schema,table_name)
    WHERE c.extra = 'auto_increment' AND t.auto_increment IS NOT NULL\G

