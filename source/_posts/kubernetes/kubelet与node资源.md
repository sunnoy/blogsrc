---
title: kubelet与node资源
date: 2019-10-10 12:22:28
tags:
- kubernetes
- node
---

# node资源

node资源不是集群的内建资源，集群中的node对象更像是实际node的一个“代理”。因此心跳机制就显得尤为重要

<!--more-->

# 心跳与租约

从1.4开始引入了Node lease对象，这个对象属于leases.coordination.k8s.io API组，并且在namespace kube-node-lease中。

lease对象是心跳的一个“代理”，为了更好的减少由于node心跳产生的master负载。

lease对象有controller manger维护，status则是有kubelet来维护。他们两个有不通的运行周期

[具体参见](https://kubernetes.io/docs/concepts/architecture/nodes/)

[设计文档](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/0009-node-heartbeat.md#graduation-criteria)

# ExternalIP

ExternalIP 是在status中的字段，一般通过云提供商来获取，作为管理员并不可以进行配置。

# kubelet的配置

kubelet后续会采用动态配置，[详细的配置字段](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/kubelet/config/v1beta1/types.go)


配置示例

```yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: xxxxxxx
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/ssl/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: cgroupfs
cgroupsPerQOS: true
clusterDNS:
- 22.68.0.2
clusterDomain: cluster.local.
configMapAndSecretChangeDetectionStrategy: Watch
containerLogMaxFiles: 3
containerLogMaxSize: 10Mi
enforceNodeAllocatable:
- pods
- kube-reserved
eventBurst: 10
eventRecordQPS: 5
evictionHard:
  imagefs.available: 15%
  memory.available: 200Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
failSwapOn: true
fileCheckFrequency: 40s
hairpinMode: hairpin-veth
healthzBindAddress: xxxxxxx
healthzPort: 10248
httpCheckFrequency: 40s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
kubeReservedCgroup: /system.slice/kubelet.service
kubeReserved: {'cpu':'200m','memory':'500Mi','ephemeral-storage':'1Gi'}
kubeAPIBurst: 100
kubeAPIQPS: 50
makeIPTablesUtilChains: true
maxOpenFiles: 1000000
maxPods: 110
nodeLeaseDurationSeconds: 40
nodeStatusReportFrequency: 1m0s
nodeStatusUpdateFrequency: 10s
oomScoreAdj: -999
podPidsLimit: -1
port: 10250
readOnlyPort: 0
resolvConf: /etc/resolv.conf
runtimeRequestTimeout: 2m0s
serializeImagePulls: true
streamingConnectionIdleTimeout: 4h0m0s
syncFrequency: 1m0s
tlsCertFile: /etc/kubernetes/ssl/kubelet.pem
tlsPrivateKeyFile: /etc/kubernetes/ssl/kubelet-key.pem
```

# APIserver认证和授权

## API server中的node认证

由于APIserver开启了客户端证书认证，kubelet访问API server的时候需要带上客户端证书。

需要注意的是，客户端证书请求中的CommonName 和 Organization 字段值在进行API server认证的时候会作为用户和用户组

APIserver会从证书中提取user name做lease的租约对象的识别对象，因此每个node的证书都是不一样的。

```json
{
  // system:node 用户组 后面接的是用户名
  "CN": "system:node:xxxxxx",
  "hosts": [
    "127.0.0.1",
    "xxxxxxx"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      // 用户组
      "O": "system:nodes",
      "OU": "System"
    }
  ]
}
```
也可以通过APIserver的参数来看

```bash
--client-ca-file string
If set, any request presenting a client certificate signed by one of the authorities in the client-ca-file is authenticated with an identity corresponding to the CommonName of the client certificate.
```

## API server中的node授权

apiserver需要启用node授权。node授权是比rbac更精细的授权策略，是专门为node这个对象适用的

[详见](https://kubernetes.io/docs/reference/access-authn-authz/node/)

```bash
--authorization-mode=Node,RBAC
```

# kubelet的认证和授权

kubelet会监听一个端口用来提供服务，比如容器日志查看以及监控指标获取。

## 认证

支持三种认证方式

```yaml
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/ssl/ca.pem
```

一般使用客户端证书认证

## 授权

### 使用webhook

kubelet有证书和webhook授权，这里主要说明webhook授权

```yaml
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
```

kubelet的webhook授权服务器是kube-apiserver，他会利用apiserver中的SubjectAccessReviewAPI来获取外部服务来请求kubelet的某些资源的具体权限

```bash
kubelet and extension API servers use this to determine user access to their own APIs.
```

### SelfSubjectAccessReview API

SelfSubjectAccessReview 在APIserver中主要负责授权，属于authorization.k8s.io API 组。包含下面的几个资源

- SubjectAccessReview - Access review for any user, not just the current one. Useful for delegating authorization decisions to the API server. For example, the kubelet and extension API servers use this to determine user access to their own APIs.
- LocalSubjectAccessReview - Like SubjectAccessReview but restricted to a specific namespace.
- SelfSubjectRulesReview - A review which returns the set of actions a user can perform within a namespace. Useful for users to quickly summarize their own access, or for UIs to hide/show actions.




