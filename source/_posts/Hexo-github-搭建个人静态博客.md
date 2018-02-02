---
title: Hexo + github 搭建个人静态博客
date: 2018-02-02 15:42:26
tags: Hexo
---

# 前言

昨天帮xkz搭了一个WordPress博客以后，我觉得应该给自己也弄一个技术博客，向大牛们看齐。

个人博客框架中最著名的莫过于WordPress，而近年来Hexo也同样流行。我对这两个框架做了比较：

- 在写作方式上，Hexo为静态博客，需要在本地完成写作后部署至服务器，仅支持Markdown；而WordPress可以在浏览器中直接写作，默认使用富文本。对于非程序员而言，后者更加友好。
- 在部署环境上，Hexo的部署只需要服务器支持读取静态文件；WordPress则需要php、MySQL等全套环境
- 在功能上，Hexo本身只提供浏览功能（但可以使用第三方扩展）；WordPress具有用户管理、评论、网站管理等相对齐全的功能
- 在数据存储与备份方式上，Hexo可以直接对md文件备份（可以使用github、百度云等方式）；WordPress需要dump数据库
- 在安全性上，Hexo仅支持静态文件访问，难以下手；而WordPress功能复杂，尤其是如果部署时使用默认参数可能引起严重的安全问题

考虑到上个月刚因为部署了XXNet client而被阿里云查水表，云主机的可靠性值得怀疑，所以我选择Hexo作为技术方案。

#  Github Page简介

[Github Page](https://pages.github.com/)是Github提供的简单建站服务，提供静态网页浏览的功能。经常见到的域名为`*.github.io`的项目介绍网站就是通过这个服务搭建的。

Github Page与名为`[username].github.io`的仓库关联。其中`[username]`为github用户名，大小写不敏感。这个仓库中的文件可以通过`https://[username].github.io/[filename]`访问。因此，我们的部署思路是：

- 在本地使用Markdown完成内容创作
- 使用Hexo输出静态网页文件（html, js, css, ...）
- 将上述静态文件上传到github仓库中
- 然后就可以通过`[username].github.io`访问博客了

在commit以后可能需要等待30s才能看到内容变化。

# Hexo的使用

## Hexo简介

Hexo是由 tommy351 开发的，基于 Node.js 的静态博客框架。**静态**主要是指博客的内容不能在浏览器中修改，而需要通过修改本地文件并重新上传的方式发表博文。

## Hexo的安装

### 安装Node.js

前往[Node.js下载页面](https://nodejs.org/zh-cn/)，下载安装程序（建议选择LTS版）。

Ubuntu/Debian用户可以直接在终端中安装：`$ sudo apt install nodejs`。

安装完成后，检查是否配置正确：`$ npm -v`，如果输出了npm的版本号，则说明安装正确。（npm是nodejs自带的包管理工具）

### 安装Hexo

1. 安装Hexo：`$ npm install hexo-cli -g` 。如果提示权限问题则使用`sudo`运行。
2. 选择一个目录作为博客目录，使用命令`$ hexo init blog`初始化博客系统
3. 切换到博客目录下安装依赖：`$ cd blog && npm install`
4. 启动预览服务器：`$ hexo s`，在浏览器中访问`localhost:4000`，如果一切正常就可以看到博客页面了

根目录下的`_config.yml`是Hexo的配置文件，可以按照个人喜好修改博客的配置。

## 添加博文

*这里只介绍最基础的文章格式，其他格式的文章请参考[官方文档](https://hexo.io/zh-cn/docs/writing.html)*

1. 新建一篇博文：`$ hexo new [title]`
2. 切换到`/source/_posts`目录下，可以看到新增了一个名为`[title].md`的文件
3. 打开`[title].md`进行创作，并保存
4. 回到根目录下，启动预览服务器(如果之前已经启动则忽略这一步) :`$ hexo s`
5. 在浏览器中浏览`localhost:4000`，可以看到新增的文章了
6. 如果对预览感到满意，则生成静态文件：`$ hexo g`

至此博客系统可以在单机上顺利运行了。但要让全世界都能看得到你的博客，还需要将它部署到github上。

## Hexo常用命令总结

- `$ hexo init [name]`: 初始化博客系统
- `$ hexo s`/`$ hexo server`：启动预览服务器。服务器启动后在浏览器中访问`localhost:4000`预览博客内容
- `$ hexo new [name]`: 新建一篇以`[name]`为标题的博文，博文保存在`/source/_posts`目录下
- `$ hexo new [layout] [name]`: 新建`layout`类型的文章
- `$ hexo g`/`$ hexo generate`：生成静态文件
- `$ hexo d`/`$ hexo deploy`: 按`_config.yml`的配置部署
- `$ hexo g -d`: 生成静态文件后部署
- `$ hexo clean`：清除静态文件缓存

# 部署与备份

## 部署方式

比较naive的部署方式为将生成的静态文件复制到仓库的本地文件夹中，再push到github仓库。

不过，Hexo提供了更加高效的部署方式：

1. 首先安装 [hexo-deployer-git](https://github.com/hexojs/hexo-deployer-git)：`$ npm install hexo-deployer-git --save`

2. 修改`_config.yml`中的`deploy`项（在文件的末尾）：

   ```yaml
   deploy:
     type: git
     repo: [repo地址]
     branch: master
   ```

3. 运行`$ hexo d`进行部署，过程中可能要求输入github的用户名和密码

每次添加或修改博文后，执行`$ hexo g -d`部署到github上。



## 备份策略

为了保障博客的数据安全，博文的Markdown源文件、Hexo配置文件、theme配置文件都应该备份。

我们可以新建一个分支用于备份这些源文件，而不必另外建一个仓库：

1. 初始化git仓库：`$ git init`
2. 新建一个分支：`$ git checkout -b hexo`
3. 添加需要管理的文件：`$ git add -A`(十分贴心的是Hexo已经提供了一个`.gitignore`文件)
4. 在本地commit：`$ git commit -m "init"`
5. 关联到github远程仓库：`$ git remote add hexo [repo地址]`
6. push到github：`$ git push`

如果需要恢复数据，只需将这个分支clone下来，并再执行一次`$ npm install`安装依赖，就能将数据（博文和设置）完全恢复了。



