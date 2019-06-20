---
title: podpreset批量添加时区配置
date: 2019-06-20 12:12:28
tags:
- kubernetes
---

用该功能来填充pod时区等，pod创建就会拥有一定的配置

<!--more-->

# 激活功能

现在还是测试阶段，对kube-apisever

```bash
#加入PodPreset
--enable-admission-plugins=PodPreset,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook \
#加入settings.k8s.io/v1alpha1=true
--runtime-config=batch/v2alpha1=true,settings.k8s.io/v1alpha1=true \
```

apiserver上面有admission-control的话，将它替换为enable-admission-plugins。admission-control即将弃用

# 使用

podpreset为namespace资源

```bash
kubectl apply -f - << EOF
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
  name: timezone
spec:
  selector:
    matchLabels:
  env:
    - name: TZ
      value: Asia/Shanghai
EOF
```