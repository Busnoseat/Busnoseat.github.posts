---
title: hexo添加可爱的live2D
date: 2020-05-27 20:20:05
tags: hexo
---
如何将可爱的2次元人物模型-live2D配置到hexo博客中来呢
<!--more-->大概只要几小步就可以完成了哦

## 安装liev2D的npm包

```
npm install --save hexo-helper-live2d

```

## 选择一款好看模型下载
```
npm install live2d-widget-model-shizuku

```
所有模型列表如下
*	live2d-widget-model-chitose
*	live2d-widget-model-epsilon2_1
*	live2d-widget-model-gf
*	live2d-widget-model-haru/01 (use npm install --save live2d-widget-model-haru)
*	live2d-widget-model-haru/02 (use npm install --save live2d-widget-model-haru)
*	live2d-widget-model-haruto
*	live2d-widget-model-hibiki
*	live2d-widget-model-hijiki
*	live2d-widget-model-izumi
*	live2d-widget-model-koharu
*	live2d-widget-model-miku
*	live2d-widget-model-ni-j
*	live2d-widget-model-nico
*	live2d-widget-model-nietzsche
*	live2d-widget-model-nipsilon
*	live2d-widget-model-nito
*	live2d-widget-model-shizuku
*	live2d-widget-model-tororo
*	live2d-widget-model-tsumiki
*	live2d-widget-model-unitychan
*	live2d-widget-model-wanko
*	live2d-widget-model-z16

## 在hexo根目录下新建文件夹live2d_models

## 在node_modules文件夹中吧下载好的模型包copy到刚创建的文件夹live2d_models中
因为我只下载了 live2d-widget-model-shizuku模型包，所以暂时先copy这一个模型。

## 在hexo根目录的配置文件_config.yml中追加以下配置
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