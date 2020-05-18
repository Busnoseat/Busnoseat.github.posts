---
title: idea创建文件夹的坑
date: 2020-02-18 19:45:36
tags: 环境
---

idea可以创建带 “.”的包名，这就埋下了一种隐患。<!--more-->什么意思呢，可以直接看下图。
![Image Text](/asset/article/20200218/1.png)
两个包名看上去完全一致，但是竟然可以共存在一个package下！这是因为这两个包根本就不是同一个包名。一个是先创建com，再创建busnoseat，再创建springcloud，...逐层依次创建的，另一个是直接创建一个package名为com.busnoseat.springcloud.test.dao.mapper。在idea里不太好区分，但是在磁盘里区别是一目了然，请看下图。
![Image Text](/asset/article/20200218/2.png)
明显看出第二种创建方式其实只是创建了一个包，只是这个包名路带有小数点而已。
线下某服务查询报错，错误提示为
```
org.apache.ibatis.binding.BindingException: Invalid bound statement (not found): com.busnoseat.springcloud.test.dao.mapper.StudentMapper.listAllStudent
```
再花了几小时确定依赖完整并且版本号一致，StudentMapper里的namespace正确，@mappserscan注解使用无误后，最终将怀疑目标移到mapperLocation上。
![Image Text](/asset/article/20200218/3.png)
看了半天还是一筹莫展的时候，直接copy了StudemtMapper文件的path粘上去然后删除多余路径，
惊奇的发现StudemtMapper所在的包名其实是<br>com.busnoseat.springcloud.test.dao.mapper/StudentMapper.xm。
不过我最终的修改方案是将包名重新逐层的创建了一遍，至此创建带小数点的包名的问题浪费了本人一下午的时间，以此为鉴，再犯剁手。

