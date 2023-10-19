---
title: hexo添加背景音乐
date: 2016-11-30 11:54:46
tags: 环境
---

hexo添加背景音乐，流程是生成iframe标签，并将标签嵌入到js里即可。
<!--more-->

# 生成iframe插件
网易云网页版里点击歌曲左下方的"生成外链播放器"，在页面里copy出已经生成好的iframe标签
![Image text](/asset/article/20190912/1.png)
![Image text](/asset/article/20190912/2.png)


# 嵌入iframe标签
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





