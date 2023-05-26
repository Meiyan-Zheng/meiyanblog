---
layout: post
title: Network architecture for controller nodes in osp-d environment
---

OSP-d is using `openshift-multus` to attach additionnal interfaces to kubevirt virt-lancher pod and providing isolated networks for openstack controller nodes. 

1. To check controller VM is running on which openshift node:
```
# oc get vmi
NAME           AGE   PHASE     IP            NODENAME          READY
controller-0   61d   Running   10.131.0.41   ostest-worker-1   True
```
From the output we can see `controller-0` is created on `worker-1`. 

2. To check NetworkAttachmentDefine in openstack namespace:
```
# oc get net-attach-def -n openstack
NAME                 AGE
ctlplane             61d
ctlplane-static      61d
external             61d
external-static      61d
internalapi          61d
internalapi-static   61d
storage              61d
storage-static       61d
storagemgmt          61d
storagemgmt-static   61d
tenant               61d
tenant-static        61d
```

3. To check the bridge used by each network:
```
# for net in `oc get net-attach-def -n openstack | grep -v NAME | grep -v static | awk '{print $1}'`;do oc get net-attach-def $net -n openstack -o yaml | egrep 'openstack.org/name:|bridge.network.kubevirt.io' ;done
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/br-ctlplane
    osp-director.openstack.org/name: ctlplane
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/br-ex
    osp-director.openstack.org/name: external
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/br-osp
    osp-director.openstack.org/name: internalapi
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/br-osp
    osp-director.openstack.org/name: storage
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/br-osp
    osp-director.openstack.org/name: storagemgmt
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/br-osp
    osp-director.openstack.org/name: tenant
```
From the output we can find ctlplane network is connected to bridge br-ctlplane, external network is connected to br-ex and internalapi/storage/storagemgmt/tenant networks are sharing bridge br-osp. 

4. Login to worker-1 and check interfaces on each bridge:
```
# oc debug node/ostest-worker-1
Starting pod/ostest-worker-1-debug ...
To use host binaries, run `chroot /host`
Pod IP: 192.168.111.21
If you don't see a command prompt, try pressing enter.
sh-4.4# chroot /host
sh-4.4# ip a | grep br-ex
4: enp6s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master br-ex state UP group default qlen 1000
38: br-ex: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
50: veth27141d1f@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-ex state UP group default 
62: veth2bc8b822@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-ex state UP group default 
sh-4.4# ip a | grep br-ctlplane
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master br-ctlplane state UP group default qlen 1000
12: br-ctlplane: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    inet 172.22.0.28/24 brd 172.22.0.255 scope global dynamic noprefixroute br-ctlplane
49: vethab0fe004@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-ctlplane state UP group default 
61: vethdf55ad06@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-ctlplane state UP group default 
sh-4.4# ip a | grep br-osp
5: enp7s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc fq_codel master br-osp state UP group default qlen 1000
11: br-osp: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc noqueue state UP group default qlen 1000
51: veth3e93b5bb@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-osp state UP group default 
63: veth6da88e10@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-osp state UP group default 
64: vethf037a556@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc noqueue master br-osp state UP group default 
65: veth84e51668@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-osp state UP group default 
66: vethd2a7bafb@if9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc noqueue master br-osp state UP group default 
```
From the output we can find each bridge is created on which interface:
```
      br-ex -- en6s0
br-ctlplane -- enp1s0
     br-osp -- enp7s0z
```
Multiple vethxxx interfaces on each bridge are those interfaces attached to kubevirt virt-launcher compute container. 


5. Login to virt-launcher pod and check mac address on each interfaces:
```
# oc rsh -n openstack virt-launcher-controller-0-hp6w8 
sh-4.4# virsh list
 Id   Name                     State
----------------------------------------
 1    openstack_controller-0   running
sh-4.4# virsh domiflist 1
 Interface   Type       Source   Model                     MAC
------------------------------------------------------------------------------
 tap0        ethernet   -        virtio-non-transitional   02:7e:90:00:00:4d
 tap1        ethernet   -        virtio-non-transitional   02:7e:90:00:00:4e
 tap2        ethernet   -        virtio-non-transitional   02:7e:90:00:00:4f
 tap3        ethernet   -        virtio-non-transitional   02:7e:90:00:00:50
 tap4        ethernet   -        virtio-non-transitional   02:7e:90:00:00:51
 tap5        ethernet   -        virtio-non-transitional   02:7e:90:00:00:52
 tap6        ethernet   -        virtio-non-transitional   02:7e:90:00:00:53
```

Check bridges created in this pod:
```
sh-4.4# ip link show type bridge
10: k6t-eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT group default 
    link/ether 02:00:00:00:00:00 brd ff:ff:ff:ff:ff:ff
12: k6t-net1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether 02:7e:90:3b:0f:7d brd ff:ff:ff:ff:ff:ff
14: k6t-net2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether 02:7e:90:c4:1a:74 brd ff:ff:ff:ff:ff:ff
16: k6t-net3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether 02:7e:90:5a:e4:54 brd ff:ff:ff:ff:ff:ff
18: k6t-net4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc noqueue state UP mode DEFAULT group default 
    link/ether 02:7e:90:e5:db:76 brd ff:ff:ff:ff:ff:ff
20: k6t-net5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether 02:7e:90:f4:b1:8f brd ff:ff:ff:ff:ff:ff
22: k6t-net6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc noqueue state UP mode DEFAULT group default 
    link/ether 02:7e:90:c8:d3:ed brd ff:ff:ff:ff:ff:ff
```
```
sh-4.4# bridge link show
4: net1@if61: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master k6t-net1 state forwarding priority 32 cost 2 
5: net2@if62: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master k6t-net2 state forwarding priority 32 cost 2 
6: net3@if63: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master k6t-net3 state forwarding priority 32 cost 2 
7: net4@if64: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 master k6t-net4 state forwarding priority 32 cost 2 
8: net5@if65: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master k6t-net5 state forwarding priority 32 cost 2 
9: net6@if66: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 master k6t-net6 state forwarding priority 32 cost 2 
11: tap0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 master k6t-eth0 state forwarding priority 32 cost 100 
13: tap1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master k6t-net1 state forwarding priority 32 cost 100 
15: tap2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master k6t-net2 state forwarding priority 32 cost 100 
17: tap3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master k6t-net3 state forwarding priority 32 cost 100 
19: tap4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 master k6t-net4 state forwarding priority 32 cost 100 
21: tap5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master k6t-net5 state forwarding priority 32 cost 100 
23: tap6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 master k6t-net6 state forwarding priority 32 cost 100 
```
We can see k6t bridges are created to provide network for virtual machine. 


6. Login to controller-0 and check mac address on each interfaces:
```
[cloud-admin@controller-0 ~]$ ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 02:7e:90:00:00:4d brd ff:ff:ff:ff:ff:ff
3: enp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 02:7e:90:00:00:4e brd ff:ff:ff:ff:ff:ff
4: enp3s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq master ovs-system state UP mode DEFAULT group default qlen 1000
    link/ether 02:7e:90:00:00:4f brd ff:ff:ff:ff:ff:ff
5: enp4s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 02:7e:90:00:00:50 brd ff:ff:ff:ff:ff:ff
6: enp5s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 02:7e:90:00:00:51 brd ff:ff:ff:ff:ff:ff
7: enp6s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether 02:7e:90:00:00:52 brd ff:ff:ff:ff:ff:ff
8: enp7s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq master ovs-system state UP mode DEFAULT group default qlen 1000
    link/ether 02:7e:90:00:00:53 brd ff:ff:ff:ff:ff:ff
9: ovs-system: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether fa:c0:20:e7:66:da brd ff:ff:ff:ff:ff:ff
10: br-ex: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 02:7e:90:00:00:4f brd ff:ff:ff:ff:ff:ff
11: br-tenant: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 02:7e:90:00:00:53 brd ff:ff:ff:ff:ff:ff
12: br-int: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 86:9d:f7:35:66:8a brd ff:ff:ff:ff:ff:ff
13: genev_sys_6081: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 65000 qdisc noqueue master ovs-system state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 42:0e:2c:43:89:a5 brd ff:ff:ff:ff:ff:ff
```
Comparing mac address, we can see interfaces are mapping to tap devices in virt-launch pod:
```
enp1s0 -- tap0
enp2s0 -- tap1
enp3s0 -- tap2
enp4s0 -- tap3
enp5s0 -- tap4
enp6s0 -- tap5
enp7s0 -- tap5
```

7. And checking network configuration on openstack controller node:
```
[root@controller-0 ~]# cat /etc/os-net-config/config.json | jq .
{
  "network_config": [
    {
      "addresses": [
        {
          "ip_netmask": "10.0.2.2/24"
        }
      ],
      "name": "nic1",
      "routes": [
        {
          "ip_netmask": "172.30.0.1/32",
          "next_hop": "10.0.2.1"
        }
      ],
      "type": "interface",
      "use_dhcp": false
    },
    {
      "addresses": [
        {
          "ip_netmask": "172.22.0.120/24"
        }
      ],
      "dns_servers": [
        "172.22.0.1"
      ],
      "domain": [
        "osptest.test.metalkube.org",
        "test.metalkube.org"
      ],
      "mtu": 1500,
      "name": "nic2",
      "routes": [],
      "type": "interface",
      "use_dhcp": false
    },
    {
      "addresses": [
        {
          "ip_netmask": "10.0.0.20/24"
        }
      ],
      "dns_servers": [
        "172.22.0.1"
      ],
      "members": [
        {
          "mtu": 1500,
          "name": "nic3",
          "primary": true,
          "type": "interface",
          "use_dhcp": false
        }
      ],
      "mtu": 1500,
      "name": "br-ex",
      "routes": [
        {
          "default": true,
          "next_hop": "10.0.0.1"
        }
      ],
      "type": "ovs_bridge",
      "use_dhcp": false
    },
    {
      "addresses": [
        {
          "ip_netmask": "172.17.0.20/24"
        }
      ],
      "mtu": 1500,
      "name": "nic4",
      "routes": [],
      "type": "interface",
      "use_dhcp": false
    },
    {
      "addresses": [
        {
          "ip_netmask": "172.18.0.20/24"
        }
      ],
      "mtu": 9000,
      "name": "nic5",
      "routes": [],
      "type": "interface",
      "use_dhcp": false
    },
    {
      "addresses": [
        {
          "ip_netmask": "172.19.0.20/24"
        }
      ],
      "mtu": 1500,
      "name": "nic6",
      "routes": [],
      "type": "interface",
      "use_dhcp": false
    },
    {
      "addresses": [
        {
          "ip_netmask": "172.20.0.20/24"
        }
      ],
      "dns_servers": [
        "172.22.0.1"
      ],
      "members": [
        {
          "mtu": 9000,
          "name": "nic7",
          "primary": true,
          "type": "interface",
          "use_dhcp": false
        }
      ],
      "mtu": 9000,
      "name": "br-tenant",
      "routes": [],
      "type": "ovs_bridge",
      "use_dhcp": false
    }
  ]
}
```
And nicx mappings are:
```
[root@controller-0 ~]# os-net-config -c /etc/os-net-config/config.json -vvv
...
[2023/05/26 01:44:59 AM] [INFO] nic5 mapped to: enp5s0
[2023/05/26 01:44:59 AM] [INFO] nic1 mapped to: enp1s0
[2023/05/26 01:44:59 AM] [INFO] nic4 mapped to: enp4s0
[2023/05/26 01:44:59 AM] [INFO] nic3 mapped to: enp3s0
[2023/05/26 01:44:59 AM] [INFO] nic7 mapped to: enp7s0
[2023/05/26 01:44:59 AM] [INFO] nic6 mapped to: enp6s0
[2023/05/26 01:44:59 AM] [INFO] nic2 mapped to: enp2s0
```

8. Now we know the network diagram will be:
<WIP on the image>

### Reference Links:
[Using the Multus CNI in OpenShift](https://cloud.redhat.com/blog/using-the-multus-cni-in-openshift) \
[Kubevirt Network Deep Dive](https://kubevirt.io/2018/KubeVirt-Network-Deep-Dive.html)
