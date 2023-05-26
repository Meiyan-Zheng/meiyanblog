---
layout: post
title: Network architecture for controller nodes in osp-d environment
---

OSP-d is using openshift-multus to attach additionnal interfaces to kubevirt virt-lancher pod and providing isolated networks for openstack controller nodes. 

1. To check controller VM is running on which openshift node:
```
[root@dell-r740-001 ~]# oc get vmi
NAME           AGE   PHASE     IP            NODENAME          READY
controller-0   61d   Running   10.131.0.41   ostest-worker-1   True
```
2.
