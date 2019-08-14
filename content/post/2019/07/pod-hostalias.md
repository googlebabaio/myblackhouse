---
title: "pod-hostalias"
date: 2019-07-01
draft: false
tags: ["k8s"]
---

利用k8s中提供的host-alias来为pod定制化域名
<!--more-->


https://kubernetes.io/docs/concepts/services-networking/add-entries-to-pod-etc-hosts-with-host-aliases/

```
apiVersion: v1
kind: Pod
...
spec:
  hostAliases:
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
...
```

这个`pod`创建后，进入相应的`pod`，查看其`hosts`可以看到如下：
```
cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1 localhost
...
10.244.135.10 hostaliases-pod
10.1.2.3 foo.remote
10.1.2.3 bar.remote

```