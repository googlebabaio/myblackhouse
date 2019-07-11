---
title: "helmV2在k8s上的安装步骤"
date: 2019-07-09
draft: false
tags: ["k8s","helm"]
---

<!--more-->

## 安装client端
```
wget https://get.helm.sh/helm-v2.14.1-linux-arm64.tar.gz
```
然后就是解压，将二进制文件放在`/usr/local/bin`


如果没有安装server(tiller)端的话,那么查看版本就只能看到client的版本号
```
[root@host-192-168-3-15 linux-amd64]# helm version
Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
Error: could not find tiller
```


## 安装tiller
利用`helm init --dry-run --debug` 查看相应初始化所需要的镜像文件
```
# helm init --dry-run --debug
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: helm
    name: tiller
  name: tiller-deploy
  namespace: kube-system
spec:
  replicas: 1
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: helm
        name: tiller
    spec:
      automountServiceAccountToken: true
      containers:
      - env:
        - name: TILLER_NAMESPACE
          value: kube-system
        - name: TILLER_HISTORY_MAX
          value: "0"
        image: gcr.io/kubernetes-helm/tiller:v2.14.1
        imagePullPolicy: IfNotPresent
        livenessProbe:
          httpGet:
            path: /liveness
            port: 44135
          initialDelaySeconds: 1
          timeoutSeconds: 1
        name: tiller
        ports:
        - containerPort: 44134
          name: tiller
        - containerPort: 44135
          name: http
        readinessProbe:
          httpGet:
            path: /readiness
            port: 44135
          initialDelaySeconds: 1
          timeoutSeconds: 1
        resources: {}
status: {}

---
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: helm
    name: tiller
  name: tiller-deploy
  namespace: kube-system
spec:
  ports:
  - name: tiller
    port: 44134
    targetPort: tiller
  selector:
    app: helm
    name: tiller
  type: ClusterIP
status:
  loadBalancer: {}
```

看到tiller需要的镜像是`gcr.io/kubernetes-helm/tiller:v2.14.1`
因为墙的原因,可以先把这个镜像下载下来然后推送到私有仓库中去
再通过`helm init`命令指定镜像名字, 安装server端 tiller
```
# helm init --upgrade --tiller-image hub.aosccs.com.cn:8888/library/tiller:v2.14.1
Creating /root/.helm/repository/repositories.yaml
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com
Adding local repo with URL: http://127.0.0.1:8879/charts
$HELM_HOME has been configured at /root/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
```

## 查看版本
```
# helm version
Client: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.14.1", GitCommit:"5270352a09c7e8b6e8c9593002a73535276507c0", GitTreeState:"clean"}
```

```
#  kubectl get pod -n kube-system -l app=helm
NAME                             READY   STATUS    RESTARTS   AGE
tiller-deploy-6dbc889b57-m59jg   1/1     Running   0          18m
```

## 添加权限
添加权限的原因是tiller默认安装在`kube-system`namespace下的,而在V2版本中,所有应用的app都是放在`kube-system`下的configmap中,所以需要一个操作其他namespace下资源的权限,在这里直接给的就是权限最大的`cluster-admin`
```
# cat rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
# kubectl create -f rbac.yaml
serviceaccount/tiller created
clusterrolebinding.rbac.authorization.k8s.io/tiller created
```

添加完权限,再把serviceaccount注入到tiller的deployment中去
```
# kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
deployment.extensions/tiller-deploy patched
```

可以把命令提示补全加上
```
# source <(helm completion bash)
```

另外，需要在每个node节点需要安装socat
```
yum install -y socat
否则，会报类似的错误：
unable to do port forwarding: socat not found.
Error: cannot connect to Tiller
```

## 常用操作
### 仓库的添加与删除
查看仓库
```
# helm repo list
NAME    URL
stable  https://kubernetes-charts.storage.googleapis.com
local   http://127.0.0.1:8879/charts
```

删除仓库
```
helm repo remove stable
```

添加仓库
```
helm repo add ali-apphub https://apphub.aliyuncs.com
```

### 安装应用
```
# helm search mysql
NAME                                    CHART VERSION   APP VERSION     DESCRIPTION
ali-apphub/mysql                        5.0.6           8.0.16          Chart to create a Highly available MySQL cluster
ali-apphub/mysqldump                    2.4.2           2.4.1           A Helm chart to help backup MySQL databases using mysqldump
ali-apphub/mysqlha                      0.5.1           5.7.13          MySQL cluster with a single master and zero or more slave...
ali-apphub/prometheus-mysql-exporter    0.3.4           v0.11.0         A Helm chart for prometheus mysql exporter with cloudsqlp...
microsoft/mysql                         1.3.0           5.7.14          Fast, reliable, scalable, and easy to use open-source rel...
microsoft/mysqldump                     2.4.2           2.4.1           A Helm chart to help backup MySQL databases using mysqldump
microsoft/prometheus-mysql-exporter     0.5.0           v0.11.0         A Helm chart for prometheus mysql exporter with cloudsqlp...
ali-apphub/percona                      1.1.0           5.7.17          free, fully compatible, enhanced, open source drop-in rep...
ali-apphub/percona-xtradb-cluster       1.0.0           5.7.19          free, fully compatible, enhanced, open source drop-in rep...
microsoft/percona                       1.1.0           5.7.17          free, fully compatible, enhanced, open source drop-in rep...
microsoft/percona-xtradb-cluster        1.0.0           5.7.19          free, fully compatible, enhanced, open source drop-in rep...
microsoft/phpmyadmin                    2.2.5           4.9.0-1         phpMyAdmin is an mysql administration frontend
ali-apphub/mariadb                      6.5.2           10.3.16         Fast, reliable, scalable, and easy to use open-source rel...
microsoft/gcloud-sqlproxy               0.6.1           1.11            DEPRECATED Google Cloud SQL Proxy
microsoft/mariadb                       6.5.5           10.3.16         Fast, reliable, scalable, and easy to use open-source rel...
# helm install -n mydb microsoft/mysql

```

### 查看应用
```
helm inspect microsoft/mysql
helm inspect values microsoft/mysql
```

### 获取应用
```
helm fetch microsoft/mysql
```
