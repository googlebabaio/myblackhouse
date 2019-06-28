---
title: "利用job来完成download的示例"
date: 2019-06-28
draft: false
tags: ["k8s"]
---

给出一个利用job来完成download的示例
<!--more-->
```
apiVersion: batch/v1
kind: Job
metadata:
  name: testdownload
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: testdownload
        command: ["wget", "-c", "https://wordpress.org/latest.zip", "-O","/tmp/peiqi11.zip"]
        image: busybox
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: download-volume
          subPath: shuju1
          mountPath: /tmp
      volumes:
      - name: download-volume
        hostPath:
          path: /tmp
          type: Directory
      nodeSelector:
        name: edge-node
```

需要注意的是`subPath`,最后真正的下载放文件存放的路径会`path`+`subpath`,即在: `/tmp/shuju1/`下
