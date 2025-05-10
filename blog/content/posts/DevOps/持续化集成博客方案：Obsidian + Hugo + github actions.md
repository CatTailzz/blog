---
title: 持续化集成博客方案：Obsidian + Hugo + github actions
subtitle: 
date: 2024-06-23T23:11:40+08:00
tags:
  - 博客搭建
categories:
  - DevOps
password: 
message: 
draft: false
---


# 需求

最近开始使用Obsidian这款笔记软件，主要看中是它的免费、拓展功能强大、本地存储。除了一些比较私人的笔记外，我更想把一些学习记录，阅读笔记等集成到博客上。正好之前接触过github actions这种自动部署发布的方式，这回打算把整个pipeline重走一遍。

在博客引擎上之前使用过vuepress和hexo，其实对于博客的需求主要是美观+简单+基础的可配置。也许是对于vuepress这种偏知识库的风格有些审美疲劳了，外加想要尝鲜，这回打算使用Hugo。大致了解了一下是用go写的，一大优势是部署很快。

# 如何实现

## 流程与方案

标准流程是先跟着指引搭建博客框架，可以在本地和github上同步一个项目，我只需要在项目的content下写我的内容就可以了。

关于内容的管理，obsidian支持按本地仓库打开一个项目进行管理，并且有git插件支持定时同步到github仓库。另一个考虑是内容中资源的管理，例如图片和音视频等，有两种选择：纯本地管理和图床管理，这里我选择了图床，优点是打包容量小，以后迁移笔记也更方便。

## 使用图床管理图片资源

图床我选择了腾讯云的COS，按天计费，也有其他方式，整体费用基本可忽略不计。

使用方式：

1. 控制台搜索对象存储进入面板
2. 在概览中，选择【创建存储桶】，进行一些简单配置，记得开启公有读
3. 安装piggo这款图床上传工具，我是M1的mac，打开时会提示已损坏，需要在命令行输入 `sudo xattr -d com.apple.quarantine /Applications/PicGo.app` 解决
4. 配置一些图床信息，其中一些密钥在【访问管理】中创建API密钥，COS存储的信息在【对象存储】-【存储桶列表】查看
5. 在Obsidian安装Image auto upload plugin插件，基本无需配置，即可在编辑时，通过拖动or粘贴方式自动上传图片。

## Hugo站点搭建

1. 访问 [Hugo的站点](https://gohugo.io/) ，根据网站指引在github上创建基础项目，并能够通过
    `hugo server` 命令在本地查看
2. 创建GitHub Pages的仓库，这里我们可以用【用户名.github.io】来命名仓库，然后把本地仓库推送到远端
3. 在github的仓库中，进入setting-Pages，选择Build and deployment的Source为github actions，然后在本地仓库中按格式创建一个workflows的配置文件，并同步到远端仓库
4. 在github的仓库中，进入actions，如果顺利则会看到一个正在运行的workflow，然后访问【用户名.github.io】这个网站就可以看到部署好的初始化Hugo站点了
5. 【可选】，使用自定义域名

## 在Obsidian中创建文章

1. 使用Obsidian打开本地仓库，进入hugo或者hugo的content文件夹，注意obsidian给每个仓库独立插件环境，所以有些要重新装。
2. 创建一篇文章需要用到hugo的命令（也可以直接新建），为了在obsidian中快速创建模板文章，我们需要用到QuickAdd这个插件
3. 在QuickAdd这款插件中，我们可以通过js脚本编写宏，也可以通过template方式更简单的配置新文件的模板，推荐js的方式，因为自定义的模板在语法上太繁琐了，用js的话就会在创建时自动使用hugo提供的模板，配置分离的方式更方便
4. 配置完成后，可以ctrl + p 唤醒操作面板，输入QuickAdd，再选择编写好的template或macro就会执行指令了

## 配置手动提交

1. 其实git插件可以设置自动提交，也可以自己手动提交，ctrl + p 唤醒的操作面板中，有git 操作可以执行。不过不习惯，还是命令行操作吧，也不要自动提交，实在太多了。
2. 如果选择自动提交，最好在commit中提交时间戳，可以方便回滚。并且时间间隔长一些，避免海量提交。修改一下测试

