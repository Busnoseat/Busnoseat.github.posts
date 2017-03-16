---
title: Linux常用命令（不定期更新）
date: 2016-12-13 17:38:06
tags: linux_环境
---

<h4>查询日志</h4>
<h5>如果日志不大</h5>
    <code> $ vim filename </code>
    <code> $ /关键字 </code>
   （查询出来后 按键n  n表示next）

<h5>根据关键字查询日志</h5>  
    <code> $ grep '关键字' filename| grep '关键字' </code>
<!--more-->

<h5>根据关键字查询日志并显示行号</h5>
    <code> $ grep -n '关键字' filename| grep -n '关键字' </code>

<h5>根据行号显示内容</h5>
    <code> $ sed -n '10,20p' catalina.out  (显示10-20行的内容) </code>
