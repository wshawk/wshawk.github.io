---
title: Github搭建Hexo双分支博客防止本地文件丢失 
date: 2022-10-18 00:28:30
categories: 
- hexo
tags: 
- hexo
top: 1
---


> 转载于： [Github搭建Hexo双分支博客防止本地文件丢失](https://www.jianshu.com/p/7d8df0de1fc7)



# 一、关于搭建流程

1. 创建仓库，Godbn.github.io

2. 创建两个分支:master和hexo

3. 在options里面设置hexo为默认分支(注意这个分支保存的是网站源文件，并不是生成后上传github上用于显示的)

4. 使用以下命令拷贝仓库到本地桌面

> git clone git@github.com:Godbn/Godbn.github.io.git

5. 终端进入Godbn.github.io目录，依次执行

>   npm install hexo
>
>   hexo init
>
>   npm install
>
>   npm install hexo-deployer-git (用于生成上传至github)

6. 修改_config.yml中的deploy参数

![img](https:////upload-images.jianshu.io/upload_images/4797359-7085b85ebbfdd97d.png?imageMogr2/auto-orient/strip|imageView2/2/w/374/format/webp)

7. 下载hexo themes主题自己百度安装调试好,我目前使用的是Sky，地址我贴出来

> https://github.com/iJinxin/hexo-theme-sky

8. 依次执行以下三条命令提交网站相关的文件

> git add .
>
> git commit -m "..."
>
> git push origin hexo

9. 执行hexo g -d 生成网站并部署到github上 (每次可以在前面输入 hexo clean 清理下)

> hexo clean
>
> hexo g -d

### 提示

**分支hexo储存的是网站的原始文件**

**分支master用来储存生成的静态网页**

# 二、日常改动

### 本地添加文章，样式等

1. 依次执行以下命令git到分支hexo上

> git add .
>
> git commit -m "..."
>
> git push origin hexo

2. 再执行以下命令发布网站到分支master

> hexo g -d

# 三、其他电脑更新github博客

1. 使用命令将仓库拷贝到本地

> git clone git@github.com:Godbn/Godbn.github.io.git

2. 在Godbn.github.io文件夹中通过以下恢复原始文件

> npm install hexo
>
> npm install
>
> npm install hexo-deployer-git --save

# 四、结束

以上是get到知乎上的回答搭建的

传送门：

> https://www.zhihu.com/question/21193762

