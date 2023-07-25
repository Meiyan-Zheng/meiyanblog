---
layout: post
title: How to enable NET_ADMIN permission to a deployment? 
---

1. Create a custom scc `my-scc`:
```
$ cat scc.yaml 
kind: SecurityContextConstraints
apiVersion: v1
metadata:
  name: my-scc
allowPrivilegeEscalation: false
allowPrivilegedContainer: false
allowedCapabilities:
  - NET_ADMIN
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: RunAsAny
fsGroup:
  type: RunAsAny
supplementalGroups:
  type: RunAsAny
users:
  - system:admin
```
2. Create a new service account `mysvcacct`:
```
$ oc create sa mysvcacct -n $NAMESPACE
```
3. Create Role and RoleBinding to connect new created SCC and SA: 
```
$ cat rbac.yaml 
---
 kind: Role
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
   name: use-scc-mysvcacct
 rules:
   - apiGroups: ["security.openshift.io"]
     resources: ["securitycontextconstraints"]
     resourceNames: ["my-scc"]
     verbs: ["use"]
---
 kind: RoleBinding
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
   name: use-scc-mysvcacct
 subjects:
   - kind: ServiceAccount
     name: mysvcacct
 roleRef:
   kind: Role
   name: use-scc-mysvcacct
   apiGroup: rbac.authorization.k8s.io
```
4. Create a deployment with `serviceAccountName` and `securityContext`:
```
$ cat simple-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: simple-deployment
    app.kubernetes.io/component: simple-deployment
    app.kubernetes.io/instance: simple-deployment
    app.kubernetes.io/part-of: simple-deployment
    app.openshift.io/runtime: redhat
  name: simple-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: simple-deployment
    type: Recreate
  template:
    metadata:
      labels:
        app: simple-deployment
        deploymentconfig: simple-deployment
    spec:
      serviceAccountName: mysvcacct
      containers:
      - image: registry.access.redhat.com/rhel7/rhel-tools:latest
        imagePullPolicy: Always
        name: simple-deployment
        command:
        - /bin/sh
        - -c
        - |
          sleep infinity
        resources: {}
        securityContext: 
          capabilities:
            add:
            - NET_ADMIN
```
5. Comfirm scc my-scc is used by the pod:
```
$ oc -n ${NAMESPACE} get pod -o 'custom-columns=NAME:metadata.name,APPLIED SCC:metadata.annotations.openshift\.io/scc'
NAME                                 APPLIED SCC
simple-deployment-5c675b55b6-6cr5t   my-scc
```
6. Confirm ip route can be edited inside of the pod:
```
$ oc rsh simple-deployment-5c675b55b6-6cr5t
sh-4.2# ip r
default via 10.128.2.1 dev eth0 
10.128.0.0/14 dev eth0 
10.128.2.0/23 dev eth0 proto kernel scope link src 10.128.3.158 
172.30.0.0/16 via 10.128.2.1 dev eth0 
224.0.0.0/4 dev eth0 
sh-4.2# ip r del 224.0.0.0/4 dev eth0 
sh-4.2# ip r add 224.0.0.0/4 dev eth0 
sh-4.2# 
```
