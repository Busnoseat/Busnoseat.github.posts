---
title: 快速删除redis缓存内容
date: 2022-11-08 10:00:00
tags: 线上问题
---

线上公司回滚过，需要快速删除redis的缓存，记下这次的步骤
<!--more-->


## 进入k8s命令行模式
![image](/asset/article/20211108/1.png)

## 连接redis
```
telnet IP地址 端口
```

## 密码登录redis
```
auth 密码
```

## 确认下目前缓存的值
```
HGET key field
```

## 删除整个key缓存
```
DEL key
```

最后附上所有redis的命令链接 http://doc.redisfans.com/