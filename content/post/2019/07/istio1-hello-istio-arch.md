# Istio的架构
先来一张istio的架构图

## 架构图
  ![](assets/markdown-img-paste-20190704142222892.png)

## 组件简要说明

- Pilot 负责在运行时配置Envoy和Mixer。

- Proxy / Envoy 每个微服务的Sidecar代理，用于处理集群中服务之间以及从服务到外部服务之间的入口/出口流量。 代理形成一个安全的微服务网格，提供丰富的功能，如发现，丰富的第7层路由，断路器，策略实施和遥测记录/报告功能。

- Mixer 在基础架构后端之上创建可移植层。 在基础架构级别实施ACL，速率限制，配额，身份验证，请求跟踪和遥测收集等策略。

- Citadel / Istio CA 通过TLS保护服务通信。 提供密钥管理系统，以自动化密钥和证书生成，分发，轮换和撤销。

- Ingress/Egress 为入站和出站外部流量配置基于路径的路由。

- Control Plane API ： 基础Orchestrator，如Kubernetes或Hashicorp Nomad。


# 安装
在k8s上的安装步骤

## 1.下载istio安装包
到 https://github.com/istio/istio/releases 去下载对应目标操作系统的安装文件
当然也可以直接下载
`curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.2.2 sh -`

进入 Istio 包目录
```
# cd istio-1.2.2/
# ls
bin  install  istio.VERSION  LICENSE  README.md  samples  tools
```

安装目录中包含：

- install/ 目录中包含了 Kubernetes 安装所需的 .yaml 文件
- samples/ 目录中是示例应用
- istioctl 客户端文件保存在 bin/ 目录之中。istioctl 的功能是手工进行 Envoy Sidecar 的注入。
- istio.VERSION 配置文件

## 2.安装 Istio 的CRD
安装:
```
for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done
```
安装完成后:
```
# kubectl get crds
NAME                                   CREATED AT
adapters.config.istio.io               2019-07-04T08:43:18Z
attributemanifests.config.istio.io     2019-07-04T08:43:18Z
authorizationpolicies.rbac.istio.io    2019-07-04T08:43:19Z
certificates.certmanager.k8s.io        2019-07-04T08:43:19Z
challenges.certmanager.k8s.io          2019-07-04T08:43:19Z
clusterissuers.certmanager.k8s.io      2019-07-04T08:43:19Z
clusterrbacconfigs.rbac.istio.io       2019-07-04T08:43:18Z
destinationrules.networking.istio.io   2019-07-04T08:43:18Z
envoyfilters.networking.istio.io       2019-07-04T08:43:18Z
gateways.networking.istio.io           2019-07-04T08:43:18Z
handlers.config.istio.io               2019-07-04T08:43:18Z
httpapispecbindings.config.istio.io    2019-07-04T08:43:18Z
httpapispecs.config.istio.io           2019-07-04T08:43:18Z
instances.config.istio.io              2019-07-04T08:43:18Z
issuers.certmanager.k8s.io             2019-07-04T08:43:19Z
meshpolicies.authentication.istio.io   2019-07-04T08:43:18Z
orders.certmanager.k8s.io              2019-07-04T08:43:19Z
policies.authentication.istio.io       2019-07-04T08:43:18Z
quotaspecbindings.config.istio.io      2019-07-04T08:43:18Z
quotaspecs.config.istio.io             2019-07-04T08:43:18Z
rbacconfigs.rbac.istio.io              2019-07-04T08:43:18Z
rules.config.istio.io                  2019-07-04T08:43:18Z
serviceentries.networking.istio.io     2019-07-04T08:43:18Z
servicerolebindings.rbac.istio.io      2019-07-04T08:43:18Z
serviceroles.rbac.istio.io             2019-07-04T08:43:18Z
sidecars.networking.istio.io           2019-07-04T08:43:19Z
templates.config.istio.io              2019-07-04T08:43:18Z
virtualservices.networking.istio.io    2019-07-04T08:43:18Z
```

## 3.安装`宽容模式`的组件和服务
所谓`宽容模式`,是这样的场景适用于以下:
- 已有应用的集群
- 注入了 Istio sidecar 的服务有和非 Istio Kubernetes 服务通信的需要
- 需要进行存活和就绪检测的应用
- Headless 服务
- StatefulSet

既然有`宽容模式`,当然也会有`严格模式`,它的适用场景:
- 这种模式会在所有的客户端和服务器之间使用 双向 TLS。
- 这种模式只适合所有工作负载都受 Istio 管理的 Kubernetes 集群。所有新部署的工作负载都会注入 Istio sidecar。
- 安装方法:`kubectl apply -f install/kubernetes/istio-demo-auth.yaml`


因为是在已有的应用的集群上安装,所以采用的是`宽容模式`
```
kubectl apply -f install/kubernetes/istio-demo.yaml
```

> 注意:不需要单独创建namespace: istio-system,yaml中已经包含有了!

安装完成过一会儿检查:
pods:
```
# kubectl get pod -n istio-system
NAME                                      READY   STATUS      RESTARTS   AGE
grafana-6fb9f8c5c7-9tnfn                  1/1     Running     0          18h
istio-citadel-68c85b6684-s4vsx            1/1     Running     0          18h
istio-cleanup-secrets-1.2.2-2wbnf         0/1     Completed   0          18h
istio-egressgateway-5f7889bf58-gvn2c      1/1     Running     0          18h
istio-galley-77d697957f-vdsxq             1/1     Running     0          18h
istio-grafana-post-install-1.2.2-6zjp4    0/1     Completed   0          18h
istio-ingressgateway-b955ddfc4-zmbbf      1/1     Running     0          18h
istio-pilot-d744c86b7-kdmsv               2/2     Running     0          18h
istio-policy-778cc8647f-7pkzp             2/2     Running     6          18h
istio-security-post-install-1.2.2-d9lmt   0/1     Completed   0          18h
istio-sidecar-injector-66549495d8-l5p7g   1/1     Running     0          18h
istio-telemetry-57c46b4f6b-82wqb          2/2     Running     6          18h
istio-tracing-5d8f57c8ff-q6zmj            1/1     Running     0          18h
kiali-7d749f9dcb-ccx7f                    1/1     Running     0          18h
prometheus-776fdf7479-2v9ph               1/1     Running     0          18h
```

svc:
```
# kubectl get svc -n istio-system
NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                                                                                      AGE
grafana                  ClusterIP   10.97.41.248     <none>        3000/TCP                                                                                                                                     20h
istio-citadel            ClusterIP   10.105.42.128    <none>        8060/TCP,15014/TCP                                                                                                                           20h
istio-egressgateway      ClusterIP   10.111.169.185   <none>        80/TCP,443/TCP,15443/TCP                                                                                                                     20h
istio-galley             ClusterIP   10.103.130.165   <none>        443/TCP,15014/TCP,9901/TCP                                                                                                                   20h
istio-ingressgateway     NodePort    10.98.7.252      <none>        15020:14258/TCP,80:31380/TCP,443:31390/TCP,31400:31400/TCP,15029:34236/TCP,15030:36304/TCP,15031:30107/TCP,15032:37550/TCP,15443:14718/TCP   20h
istio-pilot              ClusterIP   10.104.167.145   <none>        15010/TCP,15011/TCP,8080/TCP,15014/TCP                                                                                                       20h
istio-policy             ClusterIP   10.97.38.225     <none>        9091/TCP,15004/TCP,15014/TCP                                                                                                                 20h
istio-sidecar-injector   ClusterIP   10.107.7.102     <none>        443/TCP                                                                                                                                      20h
istio-telemetry          ClusterIP   10.101.129.171   <none>        9091/TCP,15004/TCP,15014/TCP,42422/TCP                                                                                                       20h
jaeger-agent             ClusterIP   None             <none>        5775/UDP,6831/UDP,6832/UDP                                                                                                                   20h
jaeger-collector         ClusterIP   10.106.186.132   <none>        14267/TCP,14268/TCP                                                                                                                          20h
jaeger-query             ClusterIP   10.104.139.212   <none>        16686/TCP                                                                                                                                    20h
kiali                    ClusterIP   10.109.83.19     <none>        20001/TCP                                                                                                                                    20h
prometheus               ClusterIP   10.103.246.130   <none>        9090/TCP                                                                                                                                     20h
tracing                  ClusterIP   10.108.248.50    <none>        80/TCP                                                                                                                                       20h
zipkin                   ClusterIP   10.105.180.170   <none>        9411/TCP                                                                                                                                     20h
```

> 需要注意的是`istio-ingressgateway`这个服务,在模板里面默认用的是lb,但是如果没有lb的环境部署的时候,`CLUSTER-IP `这一列的状态会是`pending`或者`none` ,只需要将svc的type改为`NodePort`即可。


# 参考:
https://istio.io/zh/docs/setup/kubernetes/install/kubernetes/
