---
title: Hexo 搭建博客
date: 2016-11-30 11:54:46
tags: 环境
---


#第一篇文章 当然是怎么搭建博客的


<h4>step1: 安装准备</h4>
git (这个不用说)
node (用这个来引入优秀的js框架 比如hexo)
github（自己创建的博客 最终是要挂在github上面的 ）
<!--more-->

<h4>step2: 引入hexo</h4>
新建一个文件夹 比如blog, 在文件里右键git bash
npm install -g hexo   (执行命令安装hexo)
hexo  init  (初始化)

<h4>step3: 第一次启动查看本地的博客</h4>
hexo g  (将本地文件blog下的source下的_post下的.md文件生成html)
hexo server   （本地启动hexo）
本地访问 http://localhost:4000

<h4>step4: 关联本地hexo到github</h4>
vim  _config.yml  （编辑hexo的配置文件）
翻到最下面 改成：
deploy:
     type: git
     repo: https://github.com/Busnoseat/Busnoseat.github.io.git
     branch: master
（注意 :的后面是有空格的） 

npm install hexo-deployer-git --save  （关联hexo到github上）
hexo clean  （清理之前生成的博客）
hexo g       （重新生成博客）
hexo d        (将博客推送github上)

<h4>step5：美化 复制人家的博客 哈哈</h4>
大神博客： https://github.com/litten/hexo-theme-yilia 最下面有提供copy方法

<h4>step6: 学习markdown的用法</h4> 
学习链接 ：https://www.zybuluo.com/mdeditor


最后，感谢http://www.jianshu.com/p/465830080ea9 这篇文章 文中大半参考 谢谢。





