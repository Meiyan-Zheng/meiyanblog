---
layout: post
title:  ODF upgrade - EUS to EUS
---

## Environment
- RHOCP 4.10.64
- Odf-operator.v4.10.14
- 3 masters + 3 workers
- OSD disks on worker nodes 

## Upgrade Target
- RHOCP 4.12.27
- Cluster update path: 4.10.64 -> 4.11.46 -> 4.12.27

## Reference Documentation
- [Chapter 6. Preparing to perform an EUS-to-EUS update](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.12/html/updating_clusters/preparing-eus-eus-upgrade#updating-eus-to-eus-olm-operators_eus-to-eus-upgrade)
- [Updating OpenShift Data Foundation](https://access.redhat.com/documentation/en-us/red_hat_openshift_data_foundation/4.12/html-single/updating_openshift_data_foundation/index#doc-wrapper)

## Useful Links
- [Red Hat OpenShift Container Platform Update Graph](https://access.redhat.com/labs/ocpupgradegraph/update_path)
- [Red Hat OpenShift Data Foundation Supportability and Interoperability Checker](https://access.redhat.com/labs/odfsi/#T0RGIGFzIFNlbGYtTWFuYWdlZCBTZXJ2aWNlLDQuMTAuMTIsMCwwLDAsMA==)
- [Preparing to upgrade to OpenShift Container Platform 4.12](https://access.redhat.com/articles/6955381)

## Overview Steps
- Change ocp cluster channel to eus-4.12 
- Pause the worker machine pools 
- Update RHOCP version from 4.10.64 to 4.11.46
- Update ODF version  from 4.10.14 to 4.11.9
- Update RHOCP version from 4.11.46 to 4.12.27
- Update ODF version from 4.11.9 to 4.12
- Unpause the worker machine pools  

## Detailed Commands
1. Change ocp channel to eus-4.12
```
$ oc adm upgrade channel eus-4.12
```
2. Pause the worker machine pools:
```
$ oc patch mcp/worker --type merge --patch '{"spec":{"paused":true}}'
```
3. Update RHOCP version from 4.10.64 to 4.11.46
```
$ oc adm upgrade  --to=4.11.46
```
4. Update ODF version  from 4.10.14 to 4.11.9 from Openshift Console
5. Update RHOCP version from 4.11.46 to 4.12.27
```
$ oc adm upgrade  --to=4.12.27
```
6. Update ODF version  from 4.11.9 to 4.12.6 from Openshift Console
7. Unpause the worker machine pools
```
$ oc patch mcp/worker --type merge --patch '{"spec":{"paused":false}}'
```

## Important
Since API changes between minor versions (Example 4.11.z to 4.12.z), you need to approve it manually: 
```
$ oc -n openshift-config patch cm admin-acks --patch '{"data":{"ack-4.11-kube-1.25-api-removals-in-4.12":"true"}}' --type=merge
```

## Confirm update history

We can confirm update path with below command: 
```
$ oc get clusterversion -o json|jq ".items[0].status.history"
[
  {
    "completionTime": "2023-08-24T11:55:59Z",
    "image": "quay.io/openshift-release-dev/ocp-release@sha256:e15e52f22247b833d1db59b1507fa67d920e39b75297bc3a74f3f15e560d6d02",
    "startedTime": "2023-08-24T10:48:07Z",
    "state": "Completed",
    "verified": true,
    "version": "4.12.27"
  },
  {
    "completionTime": "2023-08-24T09:11:49Z",
    "image": "quay.io/openshift-release-dev/ocp-release@sha256:88583eeaddcda4fbfdcf21f4dad86b01ff09bb010357c51f08fb24eb07fdb602",
    "startedTime": "2023-08-24T08:05:03Z",
    "state": "Completed",
    "verified": true,
    "version": "4.11.46"
  },
  {
    "completionTime": "2023-08-23T08:04:47Z",
    "image": "quay.io/openshift-release-dev/ocp-release@sha256:5b525ce48b754dc3e4119a94dee7391494934752bf98a5c352bde0b762179096",
    "startedTime": "2023-08-23T07:41:30Z",
    "state": "Completed",
    "verified": false,
    "version": "4.10.64"
  }
]
```
