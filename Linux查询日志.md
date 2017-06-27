---
title: Linux常用命令（不定期更新）
date: 2016-12-13 17:38:06
tags: linux_环境
---

# 查询日志
## 如果日志不大
    <code> $ vim filename </code>
    <code> $ /关键字 </code>
   （查询出来后 按键n  n表示next）

## 根据关键字查询日志
    <code> $ grep '关键字' filename| grep '关键字' </code>
<!--more-->

## 根据关键字查询日志并显示行号
    <code> $ grep -n '关键字' filename| grep -n '关键字' </code>

## 根据行号显示内容
    <code> $ sed -n '10,20p' catalina.out  (显示10-20行的内容) </code>


# curl调用接口

## get方法
    curl   -H "Content-type:application/json;charset=UTF-8" -X GET  http://localhost:8080/mls/loanresults?applyNo=A00451099011706230002
## post方法普通传参
    curl   -H "Content-type:application/json;charset=UTF-8" -X POST  -d  "applyNo=A00451099011706230002"  http://localhost:8080/mls/loanresults  
## post方法hjson传参
    curl   -H "Content-type:application/json;charset=UTF-8" -X POST  -d '{"applyNo":"A00451099011706230002"}'  http://localhost:8080/mls/loanresults  
