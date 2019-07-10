---
title: 复杂sql汇总
date: 2018-03-26 16:18:56
tags: sql
---

# for xml path [('code')]
将多条数据并成一条xml格式数据，并且每个字段都会成为一个元素标签。
如果code不为空并则每条数据最外层再套上一层code标签。
<!--more-->
数据库表结构如下:

|序号| hobbyID |hName| 
| :--- | :--- | :--- |
|1   |  1      | 爬山|
|2   |  2      | 游泳|
|3   |  3      | 美食|

```
SELECT * FROM @hobby FOR XML PATH('MyHobby')
运行结果如下:
<MyHobby>
  <hobbyID>1</hobbyID>
  <hName>爬山</hName>
</MyHobby>
<MyHobby>
  <hobbyID>2</hobbyID>
  <hName>游泳</hName>
</MyHobby>
<MyHobby>
  <hobbyID>3</hobbyID>
  <hName>美食</hName>
</MyHobby>
```
如果code为空则每条数据最外层不套上标签
```
SELECT * FROM @hobby FOR XML PATH('')
运行结果如下:
<hobbyID>1</hobbyID>
<hName>爬山</hName>
<hobbyID>2</hobbyID>
<hName>游泳</hName>
<hobbyID>3</hobbyID>
<hName>美食</hName>
```

也可以取出自己想要的标签并且拼装sql
```
SELECT '[ '+hName+' ]' FROM @hobby FOR XML PATH('')
运行结果如下所示:
[爬山][游泳][美食]
```

# stuff(sql,startIndex,length,param)

将sql从startIndex开始删除length长度，然后用param替代删掉的字符。注意startIndex从1开始(数据库一般是从1开始的)
```
select stuff(',[爬山],[游泳],[美食]',1,1,'')
运行结果如下: 
[爬山],[游泳],[美食]

```

# 给指定用户新增所有权限 
```
DECLARE  @columns VARCHAR(MAX), @query NVARCHAR(MAX)
SELECT @columns = stuff ((
select ','+cast(permissionId as varchar)+':允许' from org_permission for xml path(''))
,1,1,''
)
update org_user_permission_plus　
set permissioncontents=@columns  where employeeuuid='123'
```


# 查询慢sql，并且杀死慢sql

```
SELECT 'kill ' + CONVERT(nvarchar(10), session_Id), [Spid] = session_Id, ecid, [Database] = DB_NAME(sp.dbid), [User] = nt_username, [Status] = er.status, [Wait] = wait_type, [Individual Query] = SUBSTRING(qt.text, er.statement_start_offset / 2, (CASE WHEN er.statement_end_offset = - 1 THEN LEN(CONVERT(NVARCHAR(MAX), qt.text)) * 2 ELSE er.statement_end_offset END - er.statement_start_offset) / 2), [Parent Query] = qt.text, Program = program_name, Hostname, nt_domain, start_time FROM sys.dm_exec_requests er INNER JOIN sys.sysprocesses sp ON er.session_id = sp.spid CROSS APPLY sys.dm_exec_sql_text(er.sql_handle) AS qt WHERE session_Id > 50 /* Ignore system spids.a*/ AND session_Id NOT IN (@@SPID)
ORDER BY er.start_time
```


# 索引相关

```
--查询索引
select  a.name as tabname
       ,h.name as idname
from  sys.objects    as  a 
right join sys.indexes  as h  on  a.object_id=h.object_id
 where  a.type<>'s'  and a.name ='org_businessremind'

--创建索引
IF NOT EXISTS(SELECT top 1 1 from sysindexes WHERE id=object_id('org_businessremind') AND name='idx_org_businessremind_employeeId_read_deleted_businessType')
  BEGIN
		CREATE NONCLUSTERED INDEX idx_org_businessremind_employeeId_read_deleted_businessType ON dbo.org_businessremind(employeeId,[read],deleted) INCLUDE(businessType)
  END
GO

--删除索引
IF EXISTS (SELECT top 1 1 from sysindexes WHERE id=object_id('com_estate') AND name='index_com_estate_estateUuid')
BEGIN
  DROP INDEX index_com_estate_estateUuid ON dbo.com_estate
END
GO

```

# 所有公司跑一个sql

```
--创建临时表
SELECT distinct name INTO #temp  FROM  sys.databases  WHERE  state=0 AND DATABASE_id> 4

--临时表里循环跑sql
  DECLARE @dbname NVARCHAR(100)=''
  DECLARE @sql NVARCHAR(MAX)='这是一个sql'
  DECLARE @finalsql NVARCHAR(MAX)
  DECLARE @i INT
  SET @i=0
   WHILE EXISTS(SELECT * FROM  #temp)
     BEGIN
    SELECT TOP 1 @dbname=name FROM  #temp
    SET  @finalsql=N' use '+@dbname +@sql
    PRINT @finalsql
  exec sp_executesql @finalsql
  select 1
  DELETE FROM #temp WHERE  name=@dbname
  SET @i=@i+1
  PRINT @i 
 END
```
