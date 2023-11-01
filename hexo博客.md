---
title: hexo博客
date: 2016-11-30 11:54:46
tags: 环境
---

#搭建博客，添加背景音乐，添加live2D动画效果
<!--more-->

### 搭建博客

先感谢http://www.jianshu.com/p/465830080ea9 这篇文章 文中大半参考 谢谢。

* 安装准备
  git，node(用来引入js框架：hexo)，github(博客最终落地页)
* 引入hexo

```
$ npm install -g hexo   (执行命令安装hexo)
$ hexo  init  (初始化)
```

* 生成html并启动

```
$ hexo g
$ hexo s
```

* 关联本地hexo到github
  编辑hexo的配置文件_config.yml 翻到最下面 改成：

```
deploy:
type: git
repo: https://github.com/Busnoseat/Busnoseat.github.io.git
branch: master
（注意 :的后面是有空格的）
```

* 清理重新生成并push到github

```
$ npm install hexo-deployer-git --save  （关联hexo到github上）
$ hexo clean  （清理之前生成的博客）
$ hexo g       （重新生成博客）
$ hexo d        (将博客推送github上)
```

* 更换theme
  大神博客： https://github.com/litten/hexo-theme-yilia 最下面有提供copy方法

### 添加背景音乐

* 生成iframe插件
  网易云网页版里点击歌曲左下方的"生成外链播放器"，在页面里copy出已经生成好的iframe标签
  ![Image text](/asset/article/20190912/1.png)
  ![Image text](/asset/article/20190912/2.png)

* 嵌入iframe标签
  本人hexo使用的主题为next,
  修改文件blog/themes/next/layout/_macro/sidebar.swig,为了保持和原来样式一致，div最好不要自己重新写，从里面copy一个样式就行了
  ![Image text](/asset/article/20190912/3.png)

附上本人的添加的js代码

```
  <!-- 网易云音乐插件 -->
   <div class="links-of-author motion-element">
            <iframe frameborder="no" border="0" marginwidth="0" marginheight="20" width=530 height=86 src="//music.163.com/outchain/player?type=2&id=694286&auto=1&height=66"></iframe>
    </div> 
```

### 添加可爱的live2D

* 安装liev2D的npm包

```
npm install --save hexo-helper-live2d
```

* 选择一款好看模型下载

```
npm install live2d-widget-model-shizuku
```

* 在hexo根目录下新建文件夹live2d_models

* 在node_modules文件夹中吧下载好的模型包copy到刚创建的文件夹live2d_models中
因为我只下载了 live2d-widget-model-shizuku模型包，所以暂时先copy这一个模型。

* 在hexo根目录的配置文件_config.yml中追加以下配置
```
live2d:
  enable: true
  scriptFrom: local
  pluginRootPath: live2dw/
  pluginJsPath: lib/
  pluginModelPath: assets/
  tagMode: false
  debug: false
  model:
    use: live2d-widget-model-shizuku
  display:
    position: right
    width: 150
    height: 300
  mobile:
    show: true
 ```
这样就结束了，下面展示下效果图哦
![image](/asset/article/20200527/2.png)