---
title: "更换证书有效期--对于kubeadm安装的集群"
subtitile: ""
date: 2019-09-10
draft: false
tags: ["k8s"]
---

记录一下更换证书有效期的过程--对于kubeadm安装的集群
<!--more-->

kubeadm 1.14.x安装后个别证书已修改为10年有效期,但有部分证书还是一年有效期
```
/etc/kubernetes/pki/apiserver.crt
/etc/kubernetes/pki/front-proxy-ca.crt         #10年有效期
/etc/kubernetes/pki/ca.crt                     #10年有效期
/etc/kubernetes/pki/apiserver-etcd-client.crt
/etc/kubernetes/pki/front-proxy-client.crt     #10年有效期
/etc/kubernetes/pki/etcd/server.crt
/etc/kubernetes/pki/etcd/ca.crt                #10年有效期
/etc/kubernetes/pki/etcd/peer.crt              #10年有效期
/etc/kubernetes/pki/etcd/healthcheck-client.crt
/etc/kubernetes/pki/apiserver-kubelet-client.crt
```

一年过期后，会导致api service不可用，使用过程中会出现：`x509: certificate has expired or is not yet valid.` , 所以证书过期是个需要注意的事情.

# 证书说明
首先介绍下证书的使用

```
[root@host-192-168-3-15 pki]# tree
.
|-- apiserver-etcd-client.crt
|-- apiserver-etcd-client.key
|-- apiserver-kubelet-client.crt
|-- apiserver-kubelet-client.key
|-- apiserver.crt
|-- apiserver.key
|-- basic-auth.csv
|-- ca.crt
|-- ca.key
|-- etcd
|   |-- ca.crt
|   |-- ca.key
|   |-- healthcheck-client.crt
|   |-- healthcheck-client.key
|   |-- peer.crt
|   |-- peer.key
|   |-- server.crt
|   `-- server.key
|-- front-proxy-ca.crt
|-- front-proxy-ca.key
|-- front-proxy-client.crt
|-- front-proxy-client.key
|-- sa.key
`-- sa.pub
```


## Kubernetes 集群根证书

- /etc/kubernetes/pki/ca.crt
- /etc/kubernetes/pki/ca.key

由此根证书签发的证书有:

- 1. kube-apiserver 组件持有的服务端证书
  - /etc/kubernetes/pki/apiserver.crt
  - /etc/kubernetes/pki/apiserver.key

- 2. kubelet 组件持有的客户端证书
  - /etc/kubernetes/pki/apiserver-kubelet-client.crt
  - /etc/kubernetes/pki/apiserver-kubelet-client.key

> kubelet 上一般不会明确指定服务端证书, 而是只指定 ca 根证书, 让 kubelet 根据本地主机信息自动生成服务端证书并保存到配置的cert-dir文件夹中。

## 汇聚层(aggregator)证书

- /etc/kubernetes/pki/front-proxy-ca.crt
- /etc/kubernetes/pki/front-proxy-ca.key

由此根证书签发的证书只有一组:

- 1. 代理端使用的客户端证书, 用作代用户与 kube-apiserver 认证
  - /etc/kubernetes/pki/front-proxy-client.crt
  - /etc/kubernetes/pki/front-proxy-client.key

## etcd 集群根证书

- /etc/kubernetes/pki/etcd/ca.crt
- /etc/kubernetes/pki/etcd/ca.key

由此根证书签发机构签发的证书有:

- 1. etcd server 持有的服务端证书
  - /etc/kubernetes/pki/etcd/server.crt
  - /etc/kubernetes/pki/etcd/server.key

- 2. peer 集群中节点互相通信使用的客户端证书
  - /etc/kubernetes/pki/etcd/peer.crt
  - /etc/kubernetes/pki/etcd/peer.key

- 3. pod 中定义 Liveness 探针使用的客户端证书
  - /etc/kubernetes/pki/etcd/healthcheck-client.crt
  - /etc/kubernetes/pki/etcd/healthcheck-client.key

- 4. 配置在 kube-apiserver 中用来与 etcd server 做双向认证的客户端证书
  - /etc/kubernetes/pki/apiserver-etcd-client.crt
  - /etc/kubernetes/pki/apiserver-etcd-client.key

## Serveice Account秘钥
- /etc/kubernetes/pki/sa.key
- /etc/kubernetes/pki/sa.pub

这组的密钥对儿仅提供给 `kube-controller-manager` 使用. kube-controller-manager 通过 `sa.key` 对 `token` 进行签名, master 节点通过公钥 `sa.pub` 进行签名的验证.

API Server的authenticating环节支持多种身份校验方式：
- client cert
- bearer token
- static password auth

认证流程:
1. 这些方式中有一种方式通过`authenticating`（`Kubernetes API Server`会逐个方式尝试），那么身份校验就会通过。
2. 一旦API Server发现client发起的request使用的是service account token的方式，API Server就会自动采用signed bearer token方式进行身份校验。
3. 而request就会使用携带的service account token参与验证。该token是API Server在创建service account时用API server启动参数：`–service-account-key-file`的值签署(sign)生成的。
4. 如果–service-account-key-file未传入任何值，那么将默认使用`–tls-private-key-file`的值，即API Server的私钥（server.key）。
5. 通过`authenticating`后，API Server将根据Pod username所在的group：system:serviceaccounts和system:serviceaccounts:(NAMESPACE)的权限对其进行authority 和admission control两个环节的处理。在这两个环节中，cluster管理员可以对service account的权限进行细化设置。


> 注意:
> kubeadm 创建的集群, kube-proxy ,flannel,coreDNS是以 pod 形式运行的, 在 pod 中, 直接使用 service account 与 kube-apiserver 进行认证, 此时就不需要再单独为 kube-proxy 创建证书



# 更换证书过程如下:


## 查看证书过期时间
```
# openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text |grep ' Not '
            Not Before: May 24 03:31:50 2019 GMT
            Not After : May 23 03:31:50 2020 GMT
```

## 生成集群的配置文件
```
# kubeadm config view > /tmp/cluster.yaml
```


## 备份证书
```
# cp -r /etc/kubernetes/pki /etc/kubernetes/pki.bak
```

## 更新证书(适用于1.14版本及以上)
在1.14版本中,kubeadm做了改进,直接使用renew命令即可更新证书
```
# kubeadm alpha certs renew all --config=/tmp/cluster.yaml
```

更新操作会更新下面的证书
```
#-- /etc/kubernetes/pki/apiserver.key
#-- /etc/kubernetes/pki/apiserver.crt

#-- /etc/kubernetes/pki/apiserver-etcd-client.key
#-- /etc/kubernetes/pki/apiserver-etcd-client.crt

#-- /etc/kubernetes/pki/apiserver-kubelet-client.key
#-- /etc/kubernetes/pki/apiserver-kubelet-client.crt

#-- /etc/kubernetes/pki/front-proxy-client.key
#-- /etc/kubernetes/pki/front-proxy-client.crt

#-- /etc/kubernetes/pki/etcd/healthcheck-client.key
#-- /etc/kubernetes/pki/etcd/healthcheck-client.crt

#-- /etc/kubernetes/pki/etcd/peer.key
#-- /etc/kubernetes/pki/etcd/peer.crt

#-- /etc/kubernetes/pki/etcd/server.key
#-- /etc/kubernetes/pki/etcd/server.crt
```

## 更新证书(适用于1.13版本及以下)
在1.13版本及之前,需要使用`kubeadm alpha phase certs`来**生成新的证书**

### 移动老的证书
注意是: 必须移动，不然会使用现有的证书，不会重新生成!


```
cd /etc/kubernetes
mkdir -p pki.bak/etcd
mkdir conf.bak
mv pki/apiserver* ./pki.bak/
mv pki/front-proxy-client.* ./pki.bak/
mv pki/etcd/healthcheck-client.* ./pki.bak/etcd/
mv pki/etcd/peer.* ./pki.bak/etcd/
mv pki/etcd/server.* ./pki.bak/etcd/
mv ./admin.conf ./conf.bak/
mv ./kubelet.conf ./conf.bak/
mv ./controller-manager.conf ./conf.bak/
mv ./scheduler.conf ./conf.bak/
```

>注意ca的不动!


### 生成新的证书
建议不要重新生成ca证书，因为更新了ca证书，集群节点就需要手工操作，才能让集群正常(会涉及重新join)

```
kubeadm alpha phase certs etcd-healthcheck-client --config /tmp/cluster.yaml

kubeadm alpha phase certs etcd-peer --config /tmp/cluster.yaml

kubeadm alpha phase certs etcd-server --config /tmp/cluster.yaml

kubeadm alpha phase certs front-proxy-client--config /tmp/cluster.yaml

kubeadm alpha phase certs apiserver-etcd-client --config /tmp/cluster.yaml

kubeadm alpha phase certs apiserver-kubelet-client --config /tmp/cluster.yaml

kubeadm alpha phase certs apiserver --config /tmp/cluster.yaml

kubeadm alpha phase certs sa --config /tmp/cluster.yaml
```

### 更新kubeconfig文件
生成新的配置文件
```
kubeadm alpha phase kubeconfig all --apiserver-advertise-address=${MASTER_API_SERVER_IP}
```

将新生成的admin配置文件覆盖掉原本的admin文件
```
mv $HOME/.kube/config $HOME/.kube/config.old
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
sudo chmod 777 $HOME/.kube/config
```

完成后重启`kube-apiserver`,`kube-controller`,`kube-scheduler`,`etcd`这4个容器

> 如果有多台master，则将第一台生成的相关证书拷贝到其余master即可。

## 查看证书更新后的使用周期
```
# openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text |grep ' Not '
            Not Before: May 24 03:31:50 2019 GMT
            Not After : Sep  9 02:36:46 2020 GMT
```

# 参考
https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-alpha/
