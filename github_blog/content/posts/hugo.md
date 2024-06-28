---
title: "Hugo"
date: 2022-10-14T10:03:11+08:00
draft: true
---

```
指定hugo版本
hugo v0.105.0-0e3b42b4a9bdeb4d866210819fc6ddcf51582ffa+extended windows/amd64 BuildDate=2022-10-28T12:29:05Z VendorInfo=gohugoio
```



```
本地启动网站
hugo serve --disableFastRender --buildDrafts

构建网站 会生成一个 public 目录, 其中包含你网站的所有静态内容和资源. 现在可以将其部署在任何 Web 服务器上.
hugo --buildDrafts

本地创建你的第一篇文章
hugo new posts/first_post.md
```



## 重新克隆包含子模块的项目

### 第一种。递归克隆整个项目

```
videopls@f01898564faf ideaProjectsBackup % git clone git@github.com:shengtu0328/myHugo.git myHugo --recursive
Cloning into 'myHugo'...
remote: Enumerating objects: 611, done.
remote: Counting objects: 100% (20/20), done.
remote: Compressing objects: 100% (14/14), done.
remote: Total 611 (delta 4), reused 15 (delta 4), pack-reused 591
Receiving objects: 100% (611/611), 37.62 MiB | 4.30 MiB/s, done.
Resolving deltas: 100% (199/199), done.
Submodule 'github_blog/public' (git@github.com:shengtu0328/shengtu0328.github.io.git) registered for path 'github_blog/public'
Submodule 'github_blog/themes/LoveIt' (https://github.com/dillonzq/LoveIt.git) registered for path 'github_blog/themes/LoveIt'
Cloning into '/Users/videopls/ideaProjectsBackup/myHugo/github_blog/public'...
remote: Enumerating objects: 910, done.
remote: Counting objects: 100% (36/36), done.
remote: Compressing objects: 100% (24/24), done.
remote: Total 910 (delta 14), reused 20 (delta 4), pack-reused 874
Receiving objects: 100% (910/910), 39.17 MiB | 81.00 KiB/s, done.
Resolving deltas: 100% (418/418), done.
Cloning into '/Users/videopls/ideaProjectsBackup/myHugo/github_blog/themes/LoveIt'...
fatal: unable to access 'https://github.com/dillonzq/LoveIt.git/': LibreSSL SSL_connect: SSL_ERROR_SYSCALL in connection to github.com:443
fatal: clone of 'https://github.com/dillonzq/LoveIt.git' into submodule path '/Users/videopls/ideaProjectsBackup/myHugo/github_blog/themes/LoveIt' failed
Failed to clone 'github_blog/themes/LoveIt'. Retry scheduled
Cloning into '/Users/videopls/ideaProjectsBackup/myHugo/github_blog/themes/LoveIt'...
remote: Enumerating objects: 14632, done.
remote: Total 14632 (delta 0), reused 0 (delta 0), pack-reused 14632
Receiving objects: 100% (14632/14632), 45.84 MiB | 7.91 MiB/s, done.
Resolving deltas: 100% (7216/7216), done.
Submodule path 'github_blog/public': checked out '4c7446185276829a0eb6ff29c839994885163a33'
Submodule path 'github_blog/themes/LoveIt': checked out 'e9e89a4613baee823596822b7d246f5931263491'
Submodule path 'github_blog/themes/LoveIt': checked out 'e9e89a4613baee823596822b7d246f5931263491'
```

### 第二种。是先克隆父项目，再更新子模块

1. 克隆父项目

```
videopls@f01898564faf xrqProjects % git clone git@github.com:shengtu0328/myHugo.git
Cloning into 'myHugo'...
remote: Enumerating objects: 602, done.
remote: Counting objects: 100% (11/11), done.
remote: Compressing objects: 100% (9/9), done.
remote: Total 602 (delta 1), reused 6 (delta 1), pack-reused 591
Receiving objects: 100% (602/602), 37.62 MiB | 3.49 MiB/s, done.
Resolving deltas: 100% (196/196), done.
```



2. 查看子模块

```
videopls@f01898564faf myHugo % git submodule
-7bf4bb367cd252b7b99bcbf4c5bbd26aaff7d9e0 github_blog/public
-e9e89a4613baee823596822b7d246f5931263491 github_blog/themes/LoveIt
```

子模块前面有一个`-`，说明子模块文件还未检入（空文件夹）。

 

3.	初始化子模块

```
videopls@f01898564faf myHugo % git submodule init
Submodule 'github_blog/public' (git@github.com:shengtu0328/shengtu0328.github.io.git) registered for path 'github_blog/public'
Submodule 'github_blog/themes/LoveIt' (https://github.com/dillonzq/LoveIt.git) registered for path 'github_blog/themes/LoveIt'


```



4. 更新子模块

```
videopls@f01898564faf myHugo % git submodule update
Cloning into '/Users/videopls/xrqProjects/myHugo/github_blog/public'...
Cloning into '/Users/videopls/xrqProjects/myHugo/github_blog/themes/LoveIt'...
Submodule path 'github_blog/public': checked out '7bf4bb367cd252b7b99bcbf4c5bbd26aaff7d9e0'
Submodule path 'github_blog/themes/LoveIt': checked out 'e9e89a4613baee823596822b7d246f5931263491'
```



