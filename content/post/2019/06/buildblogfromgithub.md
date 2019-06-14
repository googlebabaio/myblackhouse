---
title: "hugo+github搭建自己的博客"
subtitle: 小小的记录一下
date: 2019-06-06
draft: false
tags: ["杂七杂八"]
---

记录一下用hugo+github搭建自己的博客的一些关键点
<!--more-->

# github
注册
创建仓库 `googlebabaio.github.io`

# hugo

## 安装
下载最新的hugo,解压,编辑环境变量

## 创建站点
进入到某个文件夹, 创建想要的站点名字,比如叫`lalala`
```
hugo create new lalala
```

## 新建文档
```
hugo new posts/xxx/xxx.md
```

## 下载模板
```
git clone https://github.com/coderzh/hugo-pacman-theme.git themes/myfav
```

## build文档
```
hugo --theme=beautifulhugo --baseUrl="https://googlebaba.io/"
```

## 上传静态网页到github仓库
将public下面的所有上传到github仓库`googlebabaio.github.io`


# 模板参数说明


# 注意事项
插入图片
专门拿一个仓库或者目录进行图片的保存,然后上传上去用作图床
