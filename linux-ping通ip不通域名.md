---
title: linux ping通ip不通域名
date: 2016-12-01 10:07:15
tags: linux_环境
---

<h4>step1:确定已经设置域名服务器</h4>
  $ cat /etc/resolv.conf
    -
    nameserver 8.8.8.8
    nameserver 8.8.4.4
<!--more-->
    

<h4>step2:确保网关已设置</h4>
  $ grep GATEWAY /etc/sysconfig/network-scripts/ifcfg*
    -
    /etc/sysconfig/network-scripts/ifcfg-eth0:GATEWAY=192.168.1.2

<h4>step3:确保dns设置正确</h4>
  $ grep hosts /etc/nsswitch.conf
    -
    hosts: files dns

 
 <h4>step4: 测试 </h4> 
  $ ping www.baidu.com
  -
  PING www.baidu.com (220.181.6.175) 56(84) bytes of data.
  64 bytes from 220.181.6.175: icmp_seq=0 ttl=50 time=9.51 ms   


  感谢博客： http://www.7edown.com/edu/article/soft_5126_1.html      