---
title: Redhat6.4 安装yum
date: 2016-12-01 10:17:29
tags: linux_环境
---

<h4>step1: 下载安装包</h4?
  $ uanme -r   （查看linux版本） 分i386和x86_64 <br>
     x86 地址：http://mirrors.163.com/centos/6/os/i386/Packages/  
     x86_64 地址：http://mirrors.163.com/centos/6/os/x86_64/Packages/  
     主要下载四个rpm安装包（有的版本可能依赖其他包 这时要先下载并安装依赖包） 
     python-iniparse-0.3.1-2.1.el6.noarch.rpm  
     yum-metadata-parser-1.1.2-16.el6.i686.rpm     
     yum-3.2.29-73.el6.centos.noarch.rpm  
     yum-plugin-fastestmirror-1.1.30-37.el6.noarch.rpm
     版本可能有更新 注意在下载页面关键字查询

<!--more-->     

<h4>step2: 强行卸载linux自身的yum</h4>
  $ rpm -qa | grep yum | xargs rpm -e ---nodeps
    --nodeps  强制卸载,不管依赖性

<h4>step3: 安装下载的yum包</h4>
  $ rpm -ivh python-iniparse-0.3.1-2.1.el6.noarch.rpm
  $ rpm -ivh yum-metadata-parser-1.1.2-16.el6.i686.rpm  
  $ rpm -ivh yum-3.2.29-73.el6.centos.noarch.rpm   yum-plugin-fastestmirror-1.1.30-37.el6.noarch.rpm
  [注] :最后2个需要一起安装，因为这2个互相依赖
  [注] :本人最后2个安装的时候 需要依赖python-urlgrabber-3.9.1-11.el6.noarch.rpm包 先下载安装
          $ rpm -ivh  --force python-urlgrabber-3.9.1-11.el6.noarch.rpm  强制安装

<h4>step4: 下载编辑替换帮助文档</h4>
  $ wget http://mirrors.163.com/.help/CentOS6-Base-163.repo
  $ vim CentOS6-Base-163.repo  (可以复制本文最后ps) 
  $ cp CentOS6-Base-163.repo /etc/yum.repo.d   (移动更名)     

<h4>step5: 更新yum</h4>
  $ yum clean all 清除原有缓存
  $ yum makecache  获取yum列表
    -Metadata Cache Created  (出现这个表示成功了)  

感谢： http://blog.sina.com.cn/s/blog_50f908410101cto6.html


ps 更改好的帮助文档

[base]
name=CentOS-6 - Base - 163.com
baseurl=http://mirrors.163.com/centos/6/os/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=6&arch=$basearch&repo=os
gpgcheck=1
gpgkey=http://mirror.centos.org/centos/RPM-GPG-KEY-CentOS-6
 
#released updates
[updates]
name=CentOS-6 - Updates - 163.com
baseurl=http://mirrors.163.com/centos/6/updates/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=6&arch=$basearch&repo=updates
gpgcheck=1
gpgkey=http://mirror.centos.org/centos/RPM-GPG-KEY-CentOS-6
 
#additional packages that may be useful
[extras]
name=CentOS-6 - Extras - 163.com
baseurl=http://mirrors.163.com/centos/6/extras/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=6&arch=$basearch&repo=extras
gpgcheck=1
gpgkey=http://mirror.centos.org/centos/RPM-GPG-KEY-CentOS-6
 
#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-6 - Plus - 163.com
baseurl=http://mirrors.163.com/centos/6/centosplus/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=6&arch=$basearch&repo=centosplus
gpgcheck=1
enabled=0
gpgkey=http://mirror.centos.org/centos/RPM-GPG-KEY-CentOS-6
 
#contrib - packages by Centos Users
[contrib]
name=CentOS-6 - Contrib - 163.com
baseurl=http://mirrors.163.com/centos/6/contrib/$basearch/
#mirrorlist=http://mirrorlist.centos.org/?release=6&arch=$basearch&repo=contrib
gpgcheck=1
enabled=0
gpgkey=http://mirror.centos.org/centos/RPM-GPG-KEY-CentOS-6
        

   
   
   
   
   
