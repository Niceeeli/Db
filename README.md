# Db
Common  sql 
 












mysql
查询非innodb引擎的表  
SELECT 
    table_schema,
    table_name,
    engine,
    sys.format_bytes(data_length) as data_size
FROM
    tables
WHERE
    engine <> 'InnoDB' AND table_schema NOT IN ('mysql' , 'performance_schema','information_schema'); 
    
    找出row_format 为fixed格式的表
    select table_schema,table_name,row_format,CREATE_OPTIONS from information_schema.tables where CREATE_OPTIONS like '%fix%' ;
    
  找出未使用主键的表 
    select concat(t.table_schema,'.',t.TABLE_NAME) as tablename 
From information_schema.tables t left join (Select CONSTRAINT_SCHEMA,table_name from information_schema.TABLE_CONSTRAINTS where CONSTRAINT_TYPE='PRIMARY KEY') p 
              on t.table_name=p.table_name and t.TABLE_SCHEMA=p.CONSTRAINT_SCHEMA  
Where t.table_schema not in ('performance_schema','information_schema','mysql') and p.table_name is  null ;  

SELECT
    *
FROM
    information_schema.tables t
        LEFT JOIN
    information_schema.STATISTICS s ON t.table_schema = s.table_schema
        AND t.table_name = s.table_name
        AND s.index_name = 'PRIMARY'
WHERE -- 过滤系统表，不统计视图
    t.table_schema NOT IN ('mysql' , 'performance_schema',
        'information_schema',
        'sys')
        AND table_type = 'base table'   
        AND s.index_name IS NULL;
 




查询 所有key 
select constraint_schema,table_name,constraint_name,constraint_type from 
information_schema.table_constraints where table_schema not in ('information_schema', 'mysql', 'test'); 
 


找出 自增字段 不在符合索引第一位 或者不是一个单独的key  影响 count(*) 的sql 
Select A.TABLE_SCHEMA,A.TABLE_NAME
From  (select TABLE_SCHEMA,TABLE_NAME,COLUMN_NAME From information_schema.columns where EXTRA='auto_increment' ) A,
      (select TABLE_SCHEMA,TABLE_NAME,COLUMN_NAME,ORDINAL_POSITION  From information_schema.KEY_COLUMN_USAGE   ) B
Where A.TABLE_SCHEMA=B.TABLE_SCHEMA and A.TABLE_NAME=B.TABLE_NAME and A.COLUMN_NAME=B.COLUMN_NAME 
Group by A.TABLE_SCHEMA,A.TABLE_NAME Having min(ORDINAL_POSITION) > 1 ; 

 
找出被锁者以及持有锁者的线程id 以及事务id 
SELECT     r.trx_id waiting_trx_id,     r.trx_mysql_thread_id waiting_thread,     r.trx_query  
wating_query,     b.trx_id blocking_trx_id,     b.trx_mysql_thread_id blocking_thread,     b.trx_query 
blocking_query FROM     information_schema.innodb_lock_waits w         INNER JOIN  
information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id         INNER JOIN 
information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id;   



select *From information_schema.processlist order by time desc  
select ID,USER,TIME,substring_index(HOST,':',1) as HOST,DB,COMMAND,STATE,INFO from information_schema.processlist 




















 




