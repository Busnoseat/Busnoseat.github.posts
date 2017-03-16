---
title: tomcat关闭自身的日志
date: 2016-12-02 16:33:34
tags: tomcat
---

问题： 项目日志一般用slf4j,logback等自定义日志文件，或者存到mongdb里。这样就没必要使用tomcat的打印的日志，尤其是catalina.out，虽然每天备份但是不清空自身会导致内存吃紧。

<h4>step1: 关闭日志文件</h4>
 vim  conf/logging.properties

 将level级别设置成WARNING就可以大量减少日志的输出
 一般日志的级别有： SEVERE (highest value) > WARNING > INFO > CONFIG > FINE > FINER > FINEST (lowest value)
 本人直接将有关打印的全部注视掉 这样tomcat的log下就再也不会有manage,localhost等日志文件
 <!--more-->

<h4>step2: 关闭tomcat的控制台打印</h4>

 vim  bin/catalina.sh

 查询catalina.out
 - 
 if [ -z "$CATALINA_OUT" ] ; then
　　CATALINA_OUT="$CATALINA_BASE"/logs/catalina.out
修改为
 if [ -z "$CATALINA_OUT" ] ; then
　　CATALINA_OUT=/dev/null

将控制台的语句输入到黑洞中去

<h4>step3 重启tomcat </h4>

发现log下只有 logs/juli.2016-12-02.log这一个文件， 这是tomcat启动时候的日志 而且每次启动会重新覆盖 就可以不用管它了

