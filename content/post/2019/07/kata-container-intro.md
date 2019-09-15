---
title: "介绍katacontainer"
date: 2019-07-24
draft: false
tags: ["k8s"]
---


介绍使用katacontainer
<!--more-->

![](assets/markdown-img-paste-20190915214826610.png)


![](assets/markdown-img-paste-20190915214902935.png)

# Firecracker
- Virtual Machine Monitor
- Open source by AWS - Nov 2018
  - vmm with minimal design, thus less memory footprint and  attack surface
- Supported by Kata Containers since 1.5 release


# Kubernetes RuntimeClass
- Available since kubernetes v1.12 (beta feature since v1.14)
- Kubernetes native way to define multiple container runtimes
- Supported by CRI-O and containerd


![](assets/markdown-img-paste-20190915215107530.png)

# Contribute

- https://katacontainers.io
- code/docs: https://github.com/kata-containers/
- Apache 2.0 license
- Slack: bit.ly/KataSlack
