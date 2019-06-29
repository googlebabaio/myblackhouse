---
title: "给dashboard加入CA认证"
date: 2019-06-29
draft: false
tags: ["k8s"]
---

在部署完k8s集群后,往往会部署dashborad作为基础监控的一个手段.
但有时候,部署好的dashboard在其他浏览器上无法正常访问,本文将提供解决的一个参考.
<!--more-->

1、使用APIServer的CA生成证书
```
cat > dashboard-csr.json <<EOF
{
    "CN": "Dashboard",
    "hosts": [],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing"
        }
    ]
}
EOF
```

APIServer的CA证书目录
```
K8S_CA=$1
cfssl gencert -ca=$K8S_CA/ca.pem -ca-key=$K8S_CA/ca-key.pem -config=$K8S_CA/ca-config.json -profile=kubernetes dashboard-csr.json | cfssljson -bare dashboard
```

2、 删除默认的secret，用自签证书创建新的secret
```
kubectl delete secret kubernetes-dashboard-certs -n kube-system
kubectl create secret generic kubernetes-dashboard-certs --from-file=./ -n kube-system
```

3、修改 dashboard-controller.yaml 文件，在args下面增加证书两行
```
       args:
         # PLATFORM-SPECIFIC ARGS HERE
         - --auto-generate-certificates
         - --tls-key-file=dashboard-key.pem
         - --tls-cert-file=dashboard.pem
```
最后重新提交
```
kubectl apply -f dashboard-controller.yaml
```
