---
title: '华为云CCE导出secret的值为环境变量时多了一个换行符'
tags: 公有云 云厂商 华为云 CCE Secret
key: 20180723
---

昨天晚上到今天，在上线octopus微服务时候后，发现微信接口一直不通，原因是SHA校验失败。定位了很久，可能原因都排查了:
- openjdk 与 oracle jdk差异？
- string返回的bytes因不通的字符集导致差异？
- linux与windows系统未知差异？
在排除了上述2点之后，最后一点无法排除也不可能，最后通过打印日志发现多了一个换行符。
<!--more-->
下面是secret例子，通过这个secret创建配置，然后到处环境变量到应用程序，应用程序得到的值就会多一个换行符。
```
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: secret-credentials-v0.1
data:
  accesskey: YQo=
  secretkey: Ygo=
  weixintoken: Ywo=
```