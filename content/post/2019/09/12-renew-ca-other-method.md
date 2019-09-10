---
title: "更换证书有效期的其他办法--对于kubeadm安装的集群"
subtitile: ""
date: 2019-09-10
draft: false
tags: ["k8s"]
---

更新证书除了使用kubeadm这种正常途径外,还有一些"其他方法"也可以达到目的

<!--more-->

# 方法一: 启用自动轮换 `kubelet` 证书
kubelet证书分为server和client两种， k8s 1.9默认启用了client证书的自动轮换，但server证书自动轮换需要用户开启

## 增加 controller-manager 参数
在`/etc/kubernetes/manifests/kube-controller-manager.yaml` 添加如下参数
```
  - command:
    - kube-controller-manager
    - --experimental-cluster-signing-duration=87600h0m0s #10年过期
    - --feature-gates=RotateKubeletServerCertificate=true
    - ....
```

创建 rbac 对象
创建rbac对象，允许节点轮换kubelet server证书：
```
cat > ca-update.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeserver
rules:
- apiGroups:
  - certificates.k8s.io
  resources:
  - certificatesigningrequests/selfnodeserver
  verbs:
  - create
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubeadm:node-autoapprove-certificate-server
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeserver
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:nodes
EOF

kubectl create –f ca-update.yaml
```

# 方法二: 修改源码
`staging/src/k8s.io/client-go/util/cert/cert.go`

```
const duration365d = time.Hour * 24 * 365

func NewSelfSignedCACert(cfg Config, key crypto.Signer) (*x509.Certificate, error) {
    now := time.Now()
    tmpl := x509.Certificate{
        SerialNumber: new(big.Int).SetInt64(0),
        Subject: pkix.Name{
            CommonName:   cfg.CommonName,
            Organization: cfg.Organization,
        },
        NotBefore:             now.UTC(),
        //这里已经调整为10年有效期
        NotAfter:              now.Add(duration365d * 10).UTC(),
        KeyUsage:              x509.KeyUsageKeyEncipherment | x509.KeyUsageDigitalSignature | x509.KeyUsageCertSign,
        BasicConstraintsValid: true,
        IsCA:                  true,
    }

    certDERBytes, err := x509.CreateCertificate(cryptorand.Reader, &tmpl, &tmpl, key.Public(), key)
    if err != nil {
        return nil, err
    }
    return x509.ParseCertificate(certDERBytes)
}
```
如上所示，通过NewSelfSignedCACert这个方法签发的证书都默认为10年有效期了，但这个只影响部分证书，但这样还没满足我们的需求，个别证书的有效期调整，在经过对源码的分析后，找到了如下的逻辑：

发现部分证书是通过NewSignedCert这个方法签发，而这个方法签发的证书默认只有一年有效期，查看代码逻辑：
代码:`cmd/kubeadm/app/util/pkiutil/pki_helpers.go`

```
const duration365d = time.Hour * 24 * 365

func NewSignedCert(cfg *certutil.Config, key crypto.Signer, caCert *x509.Certificate, caKey crypto.Signer) (*x509.Certificate, error) {
    serial, err := rand.Int(rand.Reader, new(big.Int).SetInt64(math.MaxInt64))
    if err != nil {
        return nil, err
    }
    if len(cfg.CommonName) == 0 {
        return nil, errors.New("must specify a CommonName")
    }
    if len(cfg.Usages) == 0 {
        return nil, errors.New("must specify at least one ExtKeyUsage")
    }

    certTmpl := x509.Certificate{
        Subject: pkix.Name{
            CommonName:   cfg.CommonName,
            Organization: cfg.Organization,
        },
        DNSNames:     cfg.AltNames.DNSNames,
        IPAddresses:  cfg.AltNames.IPs,
        SerialNumber: serial,
        NotBefore:    caCert.NotBefore,
        // 只有一年有效期
        NotAfter:     time.Now().Add(duration365d).UTC(),
        KeyUsage:     x509.KeyUsageKeyEncipherment | x509.KeyUsageDigitalSignature,
        ExtKeyUsage:  cfg.Usages,
    }
    certDERBytes, err := x509.CreateCertificate(cryptorand.Reader, &certTmpl, caCert, key.Public(), caKey)
    if err != nil {
        return nil, err
    }
    return x509.ParseCertificate(certDERBytes)
}
```
