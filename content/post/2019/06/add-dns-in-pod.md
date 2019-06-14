---
title: "给pod里的集群添加自定义DNS解析"
subtitile: "coredns的一个使用小技巧"
date: 2019-06-10
draft: false
tags: ["k8s","coredns"]
---

有时候的需求是需要集群中的pod能访问自定义的一些域名,但又不能都一一写在container里面的/etc/hosts中，这样就需要用自定义dns来实现。
coredns 自带 hosts 插件， 允许像配置 hosts 一样配置自定义 DNS 解析，修改 kube-system 中 configMap 的 coredns 添加`hosts`那一段设置即可。

<!--more-->

添加方法如下:

```
[root@host-192-168-3-15 ~]# kubectl describe cm coredns
Name:         coredns
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Data
====
Corefile:
----
.:53 {
    errors
    health
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       upstream
       fallthrough in-addr.arpa ip6.arpa
    }
    hosts {
       192.168.3.27 edgehub.acedge.cn
       192.168.3.140  static.acedge.cn

       fallthrough
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}

Events:  <none>
```
