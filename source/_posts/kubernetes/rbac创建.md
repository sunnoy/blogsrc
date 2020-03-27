---
title: rbac创建
date: 2020-03-14 12:12:28
tags:
- kubernetes
---

# 创建rbac

<!--more-->

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: service-account

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pipeline-role
rules:
- apiGroups: ["extensions", "apps", ""]
  resources: ["services", "deployments", "pods"]
  verbs: ["get", "create", "update", "patch", "list", "delete"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pipeline-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pipeline-role
subjects:
- kind: ServiceAccount
  name: service-account
  namespace: default
```