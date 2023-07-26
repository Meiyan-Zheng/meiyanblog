---
layout: post
title:  How to enable NET_ADMIN permission to a pod with custom scc?
---

1. Create custom scc:
```
$ oc whoami
system:admin
$ oc create -f scc.yaml
```
This is the yaml file to create new scc: 
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
2. Create a serviceaccount in target namespace:
```
$ oc whoami
meiyan
$ oc new-project meiyan-scc-pod
$ oc create sa meiyansvcacct
```
3. Bind the sa `meiyansvcacct` to custom scc:
```
$ oc whoami
system:admin
$ oc adm policy add-scc-to-user my-scc -z meiyansvcacct
```
4. Modify priority of scc my-scc:
```
$ oc patch scc my-scc -p '{"priority":1}' --type merge
securitycontextconstraints.security.openshift.io/my-scc patched
```
5. Create Pod in namespace:
```
apiVersion: v1
kind: Pod
metadata:
  name: simple-pod
spec:
  containers:
  - command:
    - /bin/sh
    - -c
    - |
      sleep infinity
    image: registry.access.redhat.com/rhel7/rhel-tools:latest
    imagePullPolicy: Always
    name: simple-deployment
    resources: {}
  securityContext: {}
  serviceAccount: meiyansvcacct
```
6. Confirm the new pod is using custom scc:
```
$ oc get pod -o 'custom-columns=NAME:metadata.name,APPLIED SCC:metadata.annotations.openshift\.io/scc'
NAME         APPLIED SCC
simple-pod   my-scc
```
7. WIP



 
