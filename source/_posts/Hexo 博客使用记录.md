---
title: Hexo 博客使用记录
# index_img: /img/hello_world_cover.png
comments: valine
sticky: 1001
---

终于搬新博客了

### Record 1：文章内部图片死活加载不出来

刚使用 fluid 主题时，文章封面图片可以设置，但文章内的图片无法显示，markdown 语法和 HTML 语法都无济于事。最后死马当活马医，将hexo-asset-image卸载了，重试，居然可以了。

### Record 2：图片存放问题

有两个文件夹可以存放：/source/img 和 /public/img，但图片不能放在后者中，因为 /public 目录下的东西都是执行 hexo g 生成的（从 /source 目录获取相关内容），若执行 hexo clean 就全部被删除了。

### Record 3: 引用本地文章

使用这样的格式 (明白为什么要用图片吧 -_- )：
![](/img/usage-records/cite-local-paper.png)






