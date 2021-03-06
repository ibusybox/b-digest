---
key: 20190808
title: 使用CRI-O混合编排MicroVM安全容器与LXC容器
tags: CRI-O MicroVM LXC Kata-Runtime Kata-Container 安全容器
---

在 [容器实践路线图](/2019/07/20/容器实践路线图.html) 介绍了几种应用虚拟化技术: LXC/MicroVM/LibOS ，Kata-Runtime 就是一种可以管理 MicroVM, 并兼容 OCI 与 CRI 规范的容器运行时。本文介绍如何使用 Kubernetes + CRI-O 混合编排 MicroVM容器（也称为安全容器）与LXC容器。本文的实践操作依赖[在Kubernetes中使用CRI-O运行时](/2019/08/06/在Kubernetes中使用CRI-O运行时.html)。<!--more-->

## 1. 安装Kata-Runtime

Kata-Runtime 有几种[安装模式](https://github.com/kata-containers/documentation/blob/master/install/installing-with-kata-manager.md):

- Full Installation: 安装全家桶，包含 docker/containerd 等，由于我们使用的是 CRI-O ，因此不使用这种安装方式
- Install Kata packages only: 只安装 kata 运行时，通过脚本一键安装。我们使用这种安装方式
- Manually Install: 手动安装各个部件

Install Kata packages only:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/kata-containers/tests/master/cmd/kata-manager/kata-manager.sh) install-packages"
```

安装完成之后，Kata 配置所在目录为 ```/usr/share/defaults/kata-containers/```:

```bash
# ls -l /usr/share/defaults/kata-containers
total 76
-rw-r--r-- 1 root root  9255 Jul 25 00:48 configuration-acrn.toml
-rw-r--r-- 1 root root 12902 Jul 25 00:48 configuration-fc.toml
-rw-r--r-- 1 root root 15290 Jul 25 00:48 configuration-nemu.toml
-rw-r--r-- 1 root root 15642 Jul 25 00:48 configuration-qemu.toml
-rw-r--r-- 1 root root 15577 Jul 25 00:48 configuration.toml
```
## 2. 配置 CRI-O 使用 Kata-Runtime 运行 untrused workload

CRI-O 定义了 ```trusted workload``` 与 ```untrusted workload``` ，并关联与之对应的 runtime 。
默认的所有的 workload 都是 trusted ，使用 runc 这个 runtime 。 
在 Kubernetes 的 Pod定义中通过 Pod 注解 ```io.kubernetes.cri.untrusted-workload: "true"``` 可以设定Pod为 ```untrusted workload``` ， ```unstrusted workload``` 会使用 CRI-O 配置 ```runtime_untrusted_workload``` 指定的 runtime 运行。

```bash
sed -i "s/runtime_untrusted_workload = .*/runtime_untrusted_workload = \"\/usr\/bin\/kata-runtime\"/g" /etc/crio/crio.conf
systemctl daemon-reload
systemctl restart crio
```

## 3. 运行一个 kata container

创建Pod时候增加一行注解即可将 image 运行在 kata container 中，[官方指导](https://github.com/kata-containers/documentation/blob/master/how-to/how-to-use-k8s-with-cri-containerd-and-kata.md) 中给出的注解 ```io.kubernetes.cri.untrusted-workload: "true"``` ，测试后发现不生效，CRI-O 并不会以 kata-runtime 启动容器，而是要使用 ```io.kubernetes.cri-o.TrustedSandbox: "false"``` 才可以，这个参数配置，在[虚拟机容器Kata架构](/2019/07/02/虚拟机容器Kata架构.html#cri-o-集成kata-runtime)中有详细介绍。


样例部署文件，注意 ```io.kubernetes.cri-o.TrustedSandbox: "false"``` 注解:

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1beta1","kind":"Deployment","metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},"spec":{"replicas":2,"template":{"metadata":{"labels":{"app":"nginx"}},"spec":{"containers":[{"image":"nginx:1.7.9","name":"nginx","ports":[{"containerPort":80}]}]}}}}
  labels:
    app: nginx
  name: nginx-deployment-untrusted
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      annotations:
        io.kubernetes.cri-o.TrustedSandbox: "false"
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.7.9
        imagePullPolicy: IfNotPresent
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

由于使用的是虚拟机测试的，无法安装KVM，所以最终运行报错了:

```
Events:
  Type     Reason                  Age                   From                  Message
  ----     ------                  ----                  ----                  -------
  Normal   Scheduled               14m                   default-scheduler     Successfully assigned default/nginx-deployment-untrusted-9bd997b4f-5b7vn to k8s-node-01
  Warning  FailedCreatePodSandBox  4m18s (x47 over 14m)  kubelet, k8s-node-01  Failed create pod sandbox: rpc error: code = Unknown desc = container create failed: Could not access KVM kernel module: No such file or directory
qemu-vanilla-system-x86_64: failed to initialize KVM: No such file or directory
```

在安装完 kata-runtime 之后最好使用 ```kata-runtime kata-check``` 命令检查机器是否达到运行 kata-runtime 的条件。

## Reference

[install-kata](https://github.com/kata-containers/documentation/blob/master/install/README.md)

[how-to-use-k8s-with-cri-containerd-and-kata](https://github.com/kata-containers/documentation/blob/master/how-to/how-to-use-k8s-with-cri-containerd-and-kata.md)