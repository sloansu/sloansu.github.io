---
title: Github-Hexo-Next搭建博客流程
date: 2018-06-04 18:29:48
tags:
---
本文将介绍如何大家一个属于自己的技术博客，使用Hexo管理生成你的静态博客，使用Github托管你的博客代码，使用NexT主题美化你的博客。

<!-- more -->

## Github ##

1. 首先注册一个『github』账号;
2. 建立与你的用户名对应的仓库，仓库名必须为『your_user_name.github.com』;
3. 添加SSH公钥到reposity下
  + 设置用户名和密码

			git config --global user.email
    		git config --global user.name 

  + 生成密钥：

			ssh-keygen -t rsa -C "user.email"

  + 验证绑定成功

  			ssh -T git@github.com 

## Hexo ##

1. 安装[Node.js](https://nodejs.org/en/ "Nodejs官网")。
2. 安装Git。可以使用[msysgit](http://code.google.com/p/msysgit)作为git客户端。
3. Git和Node都安装好后，可执行如下命令安装hexo。

		npm install -g hexo-cli //全局安装hexo模块
		hexo init <folder> //初始化，在指定目录中创建文件目录
		cd <folder>
		npm install //安装<folder>中的全部的依赖项,即<node_modules>
		hexo generate //生成静态页面到public/目录
		
		npm install hexo-deplorer-git --save // 安装hexo-deplorer-git
		hexo deploy //部署到远程仓库
		hexo d -g //集成上面两步的命令

执行`hexo init`后，`<folder>`中生成的目录结构和含义如下：

+ _config.yml 站点配置信息文件
+ package.json hexo博客框架模块
+ source 存放博客源文件和其他文件。`Markdown`和`HTML`文件会被解析到`Public`目录中，其他类型的文件及文件夹将会被复制到`Public`目录下。对于以`_`开头的文件及文件夹，除了`_post`文件夹外，其他以`_`开头的文件、文件夹以及隐藏文件将被忽略。
+ theme 生成页面时，根据该文件的主题来生成某个主题的页面。
+ scaffolds 生成页面时，根据该文件夹中的模板来生成也面。

4. 使用Hexo写文章

		hexo new [layout] <title> //标题中包含空格时，需要将标题用引号隐起来，
		//执行该命令后，在“source/_posts”目录下生成文件
		//通过本地文本编辑器，编辑生成的“.md”文件。

## Next ##

[NexT](http://theme-next.iissnan.com/)为Hexo的一个主题。

---

## 参考文献 ##
 [1] http://ibruce.info/2013/11/22/hexo-your-blog/