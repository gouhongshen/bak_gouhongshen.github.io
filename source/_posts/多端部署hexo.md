---
title: 多端部署 Hexo
date: 2023-06-02 13:57:18
tags:
---


文章来源：https://www.jianshu.com/p/a0824fe2e066?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation

<br>

> 害怕原文丢了，所以贴在这里

<br>

如果我想要在公司写博客怎么办，或者说如果我换电脑了怎么办，因为在 github 中的我们 github.io 项目是只有编译后的文件的，没有源文件的，也就是说，如果我们的电脑坏了，打不开了，我们的博客就不能进行更新了，所以需要把源文件也上传到 github（或者 Coding）上，然后对我们的源文件进行版本管理，这样我们就可以在另一台电脑上 pull 我们的源码，然后编译完之后 push 上去。

#### 思路
同步思路与 Github 推拉源码思路相同，使用 git 指令，保持本地的博客文件与 Github 上的博客文件相同即可
仓库分两个分支：hexo 和 master. hexo 作为默认分支，存放博客源代码，master 分支存放博客生成页面。

#### 创建 hexo 分支
默认已经有 master 分支了，创建 hexo 就可以了（名字不一定要 hexo），并将hexo设置为默认分支（Default branch）

这个 hexo 分支就是存放我们源文件的分支，我们只需要更新 hexo 分支上的内容据就好，master 上的分支 hexo 编译的时候会更新的。
配置 gitignore 文件，为了筛选出配置文件、主题目录、博文等重要信息，作为需要 GitHub 管理的文件。

public 内文件是根据 source 文件夹内容自动生成，不需要备份，不然每次改动内容太多

即使是私有仓库，除去在线服务商员工可以看到的风险外，还有云服务商被攻击造成泄漏等可能，所以不建议将配置文件传上去

```markdown
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/
_config.yml
```

初始化仓库及提交，然后我们再初始化仓库，重新对我们的代码进行版本控制

```shell
git init  //初始化本地仓库
git add -A //添加本地所有文件到仓库        
git commit -m "blog源文件" //添加commit
git branch hexo //添加本地仓库分支hexo
git remote add origin <server>  //添加远程仓库 <server> 是指在线仓库的地址 origin是本地分支,remote add操作会将本地仓库映射到云端
git push origin hexo //将本地仓库的源文件推送到远程仓库hexo分支
```

这时候博客的源文件就同步到github的hexo分支上了。

注意这里有个坑！！！如果你用的是第三方的主题 theme，是使用 git clone下来的话，要把主题文件夹下面把 .git 文件夹删除掉，不然主题无法 push 到远程仓库，导致你发布的博客是一片空白。

<br>

#### 另一台电脑的操作
首先需要搭建环境（Node，Git）

直接clone下来，修改发布后提交本地修改到远程master分支就ok.
将hexo分支clone到本地，再进入根目录安装npm

```shell
git clone <server> hexo //<server> 是指在线仓库的地址
cd hexo 
npm install
```

npm install 的时候会根据 package.json 中的插件列表自动加载相应插件。
本机的同步完成。

因为在上传博客源文件的时候忽略了配置文件（_config.yml 这是站点的配置文件）的上传，也就是没有上传配置文件的，在克隆下来的时候记得把配置文件拿过来，不然会报错。主题里面的配置文件也要（themes/next/_config.yml 这是主题配置文件）



