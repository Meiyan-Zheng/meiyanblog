---
layout: post
title: Pod stuck in terminating status
---

### Issue

Deleting namespace stuck in terminating stats:
```
# oc get ns openshift-cnv
NAME            STATUS        AGE
openshift-cnv   Terminating   35m
```

Describing namespace and it says that pod resources 
```
# oc describe ns openshift-cnv
Name:         openshift-cnv
Labels:       kubernetes.io/metadata.name=openshift-cnv
              olm.operatorgroup.uid/377da9f7-6b07-485a-8a8a-59ea65dd6555=
              openshift.io/cluster-monitoring=true
Annotations:  openshift.io/node-selector: 
              openshift.io/sa.scc.mcs: s0:c26,c25
              openshift.io/sa.scc.supplemental-groups: 1000700000/10000
              openshift.io/sa.scc.uid-range: 1000700000/10000
Status:       Terminating
Conditions:
  Type                                         Status  LastTransitionTime               Reason                  Message
  ----                                         ------  ------------------               ------                  -------
  NamespaceDeletionDiscoveryFailure            False   Mon, 27 Nov 2023 10:21:04 +0000  ResourcesDiscovered     All resources successfully discovered
  NamespaceDeletionGroupVersionParsingFailure  False   Mon, 27 Nov 2023 10:21:04 +0000  ParsedGroupVersions     All legacy kube types successfully parsed
  NamespaceDeletionContentFailure              True    Mon, 27 Nov 2023 10:21:04 +0000  ContentDeletionFailed   Failed to delete all resource types, 1 remaining: unexpected items still remain in namespace: openshift-cnv for gvr: /v1, Resource=pods
  NamespaceContentRemaining                    True    Mon, 27 Nov 2023 10:21:04 +0000  SomeResourcesRemain     Some resources are remaining: pods. has 9 resource instances
  NamespaceFinalizersRemaining                 False   Mon, 27 Nov 2023 10:21:04 +0000  ContentHasNoFinalizers  All content-preserving finalizers finished
No resource quota.
No LimitRange resource.
```

Then checking pod status in this namespace, the pods are stuck in terminating status forever:
```
# oc get pod
NAME                                                   READY   STATUS        RESTARTS   AGE
cdi-operator-5f877d475-wqr5b                           0/1     Terminating   0          10m
cluster-network-addons-operator-6c4b84976f-j29f7       0/2     Terminating   0          11m
hco-webhook-5d88499cb7-gjpcn                           0/1     Terminating   0          11m
hco-webhook-7489cdf7b-zxbb6                            0/1     Terminating   0          27m
hco-webhook-7cc697f46f-rdckt                           0/1     Terminating   0          14m
hostpath-provisioner-operator-799c685f7c-5v9d2         0/1     Terminating   0          10m
hyperconverged-cluster-cli-download-56df57c78b-7snbp   0/1     Terminating   0          27m
hyperconverged-cluster-cli-download-7c79cf4559-r2955   0/1     Terminating   0          14m
hyperconverged-cluster-cli-download-7d888645bd-25fqw   0/1     Terminating   0          11m
ssp-operator-7777c6974-mzlfs                           0/1     Terminating   0          11m
ssp-operator-8467478fd8-gcwmf                          0/1     Terminating   0          14m
tekton-tasks-operator-54f88dd876-wv9p9                 0/1     Terminating   0          14m
tekton-tasks-operator-788776b97f-jrmhv                 0/1     Terminating   0          11m
virt-operator-547867b4d9-t94jw                         0/1     Terminating   0          23m
virt-operator-64b564bd89-5ftpt                         0/1     Terminating   0          11m
virt-operator-7f988f584-rjwc4                          0/1     Terminating   0          14m
virt-operator-7f988f584-zchbs                          0/1     Terminating   0          14m
```

### Resolution

Waited more time and those pods are deleted. 
