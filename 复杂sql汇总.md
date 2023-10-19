---
title: 复杂sql汇总
date: 2018-03-26 16:18:56
tags: sql
---

记录自己平时遇到的复杂sql
<!--more-->
# 行转列(case when 或者 pivot)
接到一个需求，查询每张表的汇总数据，并且汇成一行显示

## case when的方式，先对数据分组，出参里根据某个字段进行判断
```
with 
userCountQry
as(
select cast(count(1) as VARCHAR) as num ,'userCount' as type,'companyUuid1' as name  from org_employee where deleted=0 and status in ('正式','试用','实习')  
)
,deptCountQry AS
(
select cast(count(1) as VARCHAR)  as num ,'shopCount' as type,'companyUuid1' as name  from org_department where deleted=0 and businesstype = '4' and closetime is null 
)
,firstTimeQry as(
select  CONVERT(varchar(100), min(createdtime), 112) as num,'firstUserTime' as type,'companyUuid1' as name  from org_employee  where employeeuuid <> '9999'
)
,estateQey AS(
select  cast(count(1) as VARCHAR) as num ,'estateCount' as type ,'companyUuid1' as name  from loupan_estate estate where estate.deleted=0 and ISNULL(estate.standardEstateUuid, '')<>''
)
select 
max(case when type = 'userCount' then num else 0 end) '使用经纪人量'
,max(case when type = 'shopCount' then num else 0 end) '使用门店量'
,max(case when type = 'firstUserTime' then num else 0 end) '开始使用时间'
,max(case when type = 'estateCount' then num else 0 end) '关联小区数量'
from (
select * from userCountQry
union all
select * from deptCountQry
union all
select * from firstTimeQry
union all 
select * from estateQey
)a group by a.name
```
附上查询结果：
![Image text](/asset/article/20190909/1.png)

## pivot的方式，这是sqlserver提供的一种函数，大致语法为：

```
/**
 *  大致语法为
 *  select * from sourceTable
 *  PIVOT(
 *  聚合函数（value_column）
 *  FOR pivot_column
 *  IN(<column_list>)
 *
 */

with 
userCountQry
as(
select cast(count(1) as VARCHAR) as num ,'经纪人使用量' as type  from org_employee where deleted=0 and status in ('正式','试用','实习')  
)
,deptCountQry AS
(
select cast(count(1) as VARCHAR)  as num ,'门店使用量' as type from org_department where deleted=0 and businesstype = '4' and closetime is null 
)
,firstTimeQry as(
select  CONVERT(varchar(100), min(createdtime), 112) as num,'首次使用时间' as type  from org_employee 
)
,estateQey AS(
select  cast(count(1) as VARCHAR) as num ,'小区关联使用量' as type from loupan_estate estate where estate.deleted=0 and ISNULL(estate.standardEstateUuid, '')<>''
)

select * from
(
select * from userCountQry
union all
select * from deptCountQry
union all
select * from firstTimeQry
union all 
select * from estateQey
)tb pivot(max(num) for type in(经纪人使用量,门店使用量,首次使用时间,小区关联使用量)
) pvt
```
附上查询结果：
![Image text](/asset/article/20190909/2.png)

---

# 列转行 (for xml path [('code')])
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
---

# 替换函数（stuff(sql,startIndex,length,param)）

将sql从startIndex开始删除length长度，然后用param替代删掉的字符。注意startIndex从1开始(数据库一般是从1开始的)
```
select stuff(',[爬山],[游泳],[美食]',1,1,'')
运行结果如下: 
[爬山],[游泳],[美食]

```
---

# 给指定用户新增所有权限 （列转行，再加上替换函数stuff即可）
```
DECLARE  @columns VARCHAR(MAX), @query NVARCHAR(MAX)
SELECT @columns = stuff ((
select ','+cast(permissionId as varchar)+':允许' from org_permission for xml path(''))
,1,1,''
)
update org_user_permission_plus　
set permissioncontents=@columns  where employeeuuid='123'
```
---

# 查询慢sql，并且杀死慢sql

```
SELECT 'kill ' + CONVERT(nvarchar(10), session_Id), [Spid] = session_Id, ecid, [Database] = DB_NAME(sp.dbid), [User] = nt_username, [Status] = er.status, [Wait] = wait_type, [Individual Query] = SUBSTRING(qt.text, er.statement_start_offset / 2, (CASE WHEN er.statement_end_offset = - 1 THEN LEN(CONVERT(NVARCHAR(MAX), qt.text)) * 2 ELSE er.statement_end_offset END - er.statement_start_offset) / 2), [Parent Query] = qt.text, Program = program_name, Hostname, nt_domain, start_time FROM sys.dm_exec_requests er INNER JOIN sys.sysprocesses sp ON er.session_id = sp.spid CROSS APPLY sys.dm_exec_sql_text(er.sql_handle) AS qt WHERE session_Id > 50 /* Ignore system spids.a*/ AND session_Id NOT IN (@@SPID)
ORDER BY er.start_time
```
---

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
---

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

---

# 所有公司 但是排除一部分公司 执行sql

```
-- 测试数据 指定数据库执行sql
IF EXISTS(select TOP 1 NAME
          From Master..SysDataBases
          Where DbId = (Select Dbid From Master..SysProcesses Where Spid = @@spid)
            and name not IN ('数据库名称A',"数据库名称B"))
    BEGIN
        select 1
    END
GO
```

---


# 游标
如果需要循环处理sql，并且数据量不是很大的时候，可以使用游标。

```

declare   @id int           
declare   @name varchar(50)   

--创建一个游标
declare my_cursor cursor for     
select id,name from my_user 

--打开游标
open my_cursor   
  
--获取下一条数据并赋值给变量
fetch next from my_cursor into @id,@name  

--while循环判断
while @@FETCH_STATUS=0 
begin
print(@name) --print()
select * from my_user where id=@id 
--这里一定要在while循环里手动转至下一条数据
fetch next from my_cursor into @id,@name 
end

--关闭游标
close my_cursor
--删除游标
deallocate my_cursor


```
