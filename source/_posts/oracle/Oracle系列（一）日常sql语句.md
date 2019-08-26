---
layout: post
title: Oracle系列（一）日常sql语句
date: 2019-8-17 11:25:56
categories: blog
tags: [oracle]
description: 日常语句记录 oracle
---



### Oracle系列（一）日常sql语句

##### 查看表空间大小

<!-- more -->

```sql
select b.file_id　　文件ID,
　　b.tablespace_name　　表空间,
　　b.file_name　　　　　物理文件名,
　　b.bytes/1024/1024/1024　　　　　　　总字节数(GB),
　　(b.bytes-sum(nvl(a.bytes,0)))　　　已使用,
　　sum(nvl(a.bytes,0))　　　　　　　　剩余,
　　sum(nvl(a.bytes,0))/(b.bytes)*100　剩余百分比
from dba_free_space a,dba_data_files b
where a.file_id=b.file_id
group by b.tablespace_name,b.file_name,b.file_id,b.bytes
order by b.tablespace_name
```



##### 查看表空间（方式二）

```sql
--1G=1024MB 
--1M=1024KB 
--1K=1024Bytes 
--1M=11048576Bytes 
--1G=1024*11048576Bytes=11313741824Bytes 
SELECT a.tablespace_name "表空间名", 
        total "表空间大小", 
        free "表空间剩余大小", 
        (total - free) "表空间使用大小", 
        total / (1024 * 1024 * 1024) "表空间大小(G)", 
        free / (1024 * 1024 * 1024) "表空间剩余大小(G)", 
        (total - free) / (1024 * 1024 * 1024) "表空间使用大小(G)", 
        round((total - free) / total, 4) * 100 "使用率 %" 
FROM (SELECT tablespace_name, SUM(bytes) free 
FROM dba_free_space 
GROUP BY tablespace_name) a, 
(SELECT tablespace_name, SUM(bytes) total 
FROM dba_data_files 
GROUP BY tablespace_name) b 
WHERE a.tablespace_name = b.tablespace_name
```



##### 查看版本号

```sql
select * from v$version;
```



##### 查看可能出现oracle ORA-01461 错误 can bind a LONG value only for insert into a LONG column的表

```sql
SELECT * FROM 
(SELECT TABLE_NAME, OWNER, count(*) NUM 
FROM DBA_TAB_COLUMNS 
WHERE DATA_TYPE='LONG' 
OR (( DATA_TYPE='VARCHAR2' 
or DATA_TYPE='CHAR' 
or DATA_TYPE='NVARCHAR2' 
or DATA_TYPE='NCHAR') 
AND DATA_LENGTH > 1333) 
AND OWNER NOT IN 
('SYS','SYSTEM','SH','OLAPSYS','MDSYS','WKSYS','ODM','XDB','WMSYS') 
GROUP BY TABLE_NAME, OWNER) 
WHERE NUM > 1;
```



##### user_tables(当前用户下所有的表)

```sql
select count(*) from user_tables;
```



##### 查看所有用户

```sql
select distinct owner from all_objects;
```



##### 查看当前用户表空间

```sql
select default_tablespace from dba_users where username='PINGAN_OPENAPI';
```



##### 授予LNCHDPSQP可以查看SYNCHRO_PRODUCE_TIME_TEST

```sql
grant select on SYNCHRO_PRODUCE_TIME_TEST to LNCHDPSQP;
```



##### 清空表

```sql
truncate table **
```



##### 查看连接数

```sql
select count(*) from v$session;
```



##### 大数据批量更新

```sql
declare
 cursor cur is--声明游标cur
 select B."big_name", A."UUID" ROW_ID
  FROM STATISTICS_REAL_TIME_ANALYSIS A ,"HCICLOUD"."big_small_temp" B
  WHERE A."KEY"=B."small_name"
  ORDER BY A."UUID";--从A和B表中找到ID对应的openid，并对游标内数组排序
 V_COUNTER NUMBER;--声明一个number类型的变量
BEGIN 
V_COUNTER:=0;--初始化变量值为0
 FOR ROW IN CUR LOOP--遍历游标
  UPDATE STATISTICS_REAL_TIME_ANALYSIS A SET A."BIG_KEY"=ROW."big_name" WHERE A."UUID"=ROW.ROW_ID;
    V_COUNTER:=V_COUNTER+1;--每次循环变量值+1
  IF(V_COUNTER>=1000) THEN
    COMMIT;
    V_COUNTER:=0;--每更新1000行，V_COUNTER值为1000时候，就提交给数据库,提交后将变量归零，继续下一个1000行更新
  END IF;
 END LOOP;
  COMMIT;
END;

注意：关联字段A."KEY" 和 A."UUID" 一定要加索引
测试结果：152w(8s)

单表更新
declare
 cursor cur is--声明游标cur
 SELECT UUID FROM SEND_SUCCESS_LOG;
 V_COUNTER NUMBER;--声明一个number类型的变量
BEGIN 
V_COUNTER:=0;--初始化变量值为0
 FOR ROW IN CUR LOOP--遍历游标
  UPDATE SEND_SUCCESS_LOG A SET A."VERSION"=0 WHERE A."UUID"=ROW.UUID;
    V_COUNTER:=V_COUNTER+1;--每次循环变量值+1
  IF(V_COUNTER>=1000) THEN
    COMMIT;
    V_COUNTER:=0;--每更新1000行，V_COUNTER值为1000时候，就提交给数据库,提交后将变量归零，继续下一个1000行更新
  END IF;
 END LOOP;
  COMMIT;
END;
```



##### 查看USERS表空间的全部索引，同时排序

```sql
select segment_name,tablespace_name,bytes B, bytes/1024 KB, bytes/1024/1024 MB from user_segments where segment_type='INDEX' and tablespace_name='USERS' ORDER BY bytes desc;
```



##### 查看USERS表空间的全部表，同时排序

```sql
select segment_name,tablespace_name,bytes B, bytes/1024 KB, bytes/1024/1024 MB from user_segments where segment_type='TABLE' and tablespace_name='USERS' ORDER BY bytes desc;
```



##### 查看表所在的表空间

```sql
select tablespace_name,table_name  from user_tables where table_name='STATISTICS_REAL_TIME_ANALYSIS';
```



##### 查看USERS表空间下的表

```sql
select TABLE_NAME,TABLESPACE_NAME from dba_tables where TABLESPACE_NAME='USERS';
```



##### 查看STATISTICS_REAL_TIME_ANALYSIS表占用的大小

```sql
select t.segment_name, t.segment_type, sum(t.bytes / 1024 / 1024) "占用空间(M)"
from dba_segments t
where t.segment_type='TABLE'
and t.segment_name='STATISTICS_REAL_TIME_ANALYSIS'
group by OWNER, t.segment_name, t.segment_type;
```



##### 查看USERS表空间下的表的大小

```sql
select segment_name,tablespace_name,bytes B, bytes/1024 KB, bytes/1024/1024 MB 
from user_segments 
where segment_type='TABLE' and tablespace_name='USERS' 
ORDER BY bytes desc;
```



##### 查看TS_JIETONG_INFO表空间下索引大小

```sql
select segment_name,tablespace_name,bytes B, bytes/1024 KB, bytes/1024/1024 MB 
from user_segments 
where segment_type='INDEX' and tablespace_name='TS_JIETONG_INFO' 
ORDER BY bytes desc;
```



##### 迁移索引‘INDEX_JANUARY_CALL_ID’到新表空间

```sql
alter index INDEX_JANUARY_CALL_ID rebuild tablespace TS_JIETONG_INFO;
```



##### 迁移表‘STATISTICS_TA_ANALYSIS’到新表空间

```sql
alter table STATISTICS_TA_ANALYSIS move tablespace TS_JIETONG_INFO;
```



##### 查看索引状态

```sql
SELECT OWNER, INDEX_NAME,STATUS  FROM ALL_INDEXES WHERE INDEX_NAME='INDEX_APR';
```



##### 新建索引

```sql
create index HW_AGENTINFO_AGENTID on T_HW_AGENTINFO(AGENTID) tablespace TS_JIETONG_INFO;
create index INDEX_APRIL_AGENTID on CALL_APRIL_HISTORY(AGENTID,CALL_PHONE) tablespace TS_JIETONG_INFO;
```



##### case-when 使用

```sql
SELECT
	MOD (TO_CHAR(SYSDATE, 'ss'), 2) AS i,
	COUNT (*)
FROM
	SEND_ACCESS_LOG
WHERE
	OPEN_API_CREATE_TIME BETWEEN TO_DATE (
		SYSDATE -- 			'2019-06-04 00:00:00',
		-- 			'yyyy-mm-dd hh24:mi:ss'
	)
AND TO_DATE (
	SYSDATE + 1 -- 		'2019-06-05 00:00:00',
	-- 		'yyyy-mm-dd hh24:mi:ss'
)
AND (
	(
		MOD (TO_CHAR(SYSDATE, 'ss'), 2) = '0'
		AND SUCCESSSEND_CODE = '0'
	)
	OR (
		MOD (TO_CHAR(SYSDATE, 'ss'), 2) = '1' -- 		AND SUCCESSSEND_CODE IS NULL
	)
) -- (
-- 	CASE
-- 	WHEN MOD (TO_CHAR(SYSDATE, 'ss'), 2) = '0' THEN
-- 		'0'
-- 	ELSE
-- 		null
-- 	END
-- )
```



##### 按照时间月份分组查询

```sql
SELECT
	TO_CHAR (CREATE_TIME, 'yyyy-mm') time,
	COUNT (*)
FROM
	STATISTICS_REAL_TIME_ANALYSIS
GROUP BY
	TO_CHAR (CREATE_TIME, 'yyyy-mm')
ORDER BY
	time;

```





