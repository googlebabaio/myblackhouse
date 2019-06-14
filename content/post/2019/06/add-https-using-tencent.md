---
title: "centos7.4上给apache加上https的步骤"
date: 2019-06-11
draft: false
tags: ["linux"]
---

现在一般网站都要加上https访问,才让人有安全,所以本文记录一下怎么给自己的网站加上https安全证书的
<!--more-->

# centos7.4上给apache加上https的步骤

## 1.申请ssl证书
我是去腾讯上申请的免费的DV证书,最后下载下来的证书是这样的

![](../assets/markdown-img-paste-20181218135943245.png)

打开apache的

![](../assets/markdown-img-paste-20181218140006885.png)

将这3个证书传到服务器的 /etc/httpd/ssl目录

## 2.安装mod_ssl
yum install -y mod_ssl

## 3.修改ssl的配置
vim /etc/httpd/conf.d/ssl.conf
```
DocumentRoot "/var/www/html"
ServerName www.googlebaba.io:443

SSLEngine on

SSLCertificateFile /etc/httpd/ssl/2_www.googlebaba.io.crt
SSLCertificateKeyFile /etc/httpd/ssl/3_www.googlebaba.io.key
SSLCertificateChainFile /etc/httpd/ssl/1_root_bundle.crt
```

## 4.重启服务并验证
```
systemctl restart httpd
```
