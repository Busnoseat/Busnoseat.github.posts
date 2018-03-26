---
title: 复杂sql汇总
date: 2018-03-26 16:18:56
tags: sql
---


记录平时用到的复杂sql:
[for xml path](#1)
[stuff](#2)
[for xml path+stuff](#3)
<!--more-->

<span id = "1"><h3>for xml path [('code')]</h3></span>
将多条数据并成一条xml格式数据，并且每个字段都会成为一个元素标签。
如果code不为空并则每条数据最外层再套上一层code标签。
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

<span id = "2"><h3>stuff(sql,startIndex,length,param)</h3></span>

将sql从startIndex开始删除length长度，然后用param替代删掉的字符。注意startIndex从1开始(数据库一般是从1开始的)
```
select stuff(',[爬山],[游泳],[美食]',1,1,'')
运行结果如下: 
[爬山],[游泳],[美食]

```

<span id = "3"><h3>给指定用户新增所有权限</h3></span>
```
DECLARE  @columns VARCHAR(MAX), @query NVARCHAR(MAX)
SELECT @columns = stuff ((
select ','+cast(permissionId as varchar)+':允许' from org_permission for xml path(''))
,1,1,''
)
update org_user_permission_plus　
set permissioncontents=@columns  where employeeuuid='123'
```

