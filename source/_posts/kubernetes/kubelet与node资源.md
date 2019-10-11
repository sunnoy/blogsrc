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
- 10.68.0.2
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