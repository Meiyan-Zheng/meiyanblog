---
layout: post
title: More details about PTP on OpenShift 
---

## A high-level synchronization configuration using OpenShift

<img width="791" alt="Screenshot 2023-04-20 at 15 22 54" src="https://user-images.githubusercontent.com/30589773/233291005-f040624e-a267-4baa-bfb4-2203bdd64288.png">

In this diagram, we have different type of PTP clocks. 
* Grandmaster (GM) clock: This is the primary reference time clock (PRTC) for the entire PTP network. It usually synchronizes its clock from an external Global Navigation Satellite System (GNSS) source.
* Boundary clock (BC): This intermediate device has multiple PTP-capable network connections to synchronize one network segment to another accurately. It synchronizes its clock to a master and serves as a time source for ordinary clocks.
* Ordinary clock (OC): By contrast to boundary clocks, this device only has a single PTP-capable network connection. Its main function is to synchronize its clock to a master, and in the event of losing its master, it may tolerate a loss of sync source for some period of time.

## PTP Operator 

<img width="794" alt="Screenshot 2023-04-20 at 15 33 06" src="https://user-images.githubusercontent.com/30589773/233293355-70623622-359b-4364-97ce-6c084b906f3d.png">

PTP operator offers three Custom Resource Definition (CRD):
* `NodePtpDevice`: It is responsible for discovering PTP-capable network devices in the cluster nodes.
* `PtpConfig`: Directly interact with the linuxptp processes to properly apply PTP configurations.
* `PtpOperatorConfig`: Directly interact with the linuxptp processes to properly apply PTP configurations as well as PtpConfig

The `linuxptp` software is a PTP implementation compliant with the IEEE standard 1588. 
It includes:
* `ptp4l`: Main daemon for PTP. It can be configured to act as Grandmaster, boundary clock or ordinary clock. It performs NIC clock synchronization over the network
* `phc2sys`: It container is responsible for synchronizing the two available clocks in a cluster node, typically these are the PHC and the system clocks. This program is used when hardware time stamping is configured.
* `tsc2phc`: Synchronizes one or more PTP Hardware Clocks using external time stamps. It is used to synchronize a NIC clock using an external source for time stamps. 


## High-level synchronization process in SNO clusters using the PTP operator

<img width="802" alt="Screenshot 2023-04-20 at 16 52 49" src="https://user-images.githubusercontent.com/30589773/233313468-95826e08-edc9-443e-acc2-1aba411e6ebc.png">

#### Synchronization Process

1. When receiving PTP messages, the PHC at the cluster nodes will first set the timestamps using a hardware timestamp unit (TSU) present in the receiving NIC. 
2. Once that operation is completed, the ptp4l then uses a servo algorithm to adjust the phase (synchronization) and frequency (syntonization) of the underlying PHC.
3. Once PTP port roles are determined, whether master or slave, the BC's master ports are used to synchronize the PHCs belonging to NIC slave ports of downstream nodes.
4. At the same time, the phc2sys process running in both the O-DU and O-RU nodes will read the PHC timestamps from the ptp4l program through a shared Unix Domain Socket (UDS) and finally adjust the SYSTEM_CLOCK offset.



## Reference Links:
[0] [https://cloud.redhat.com/blog/delivering-high-precision-clock-synchronization-for-low-latency-5g-networks-with-openshift-part-1](https://cloud.redhat.com/blog/delivering-high-precision-clock-synchronization-for-low-latency-5g-networks-with-openshift-part-1) \
[1] [https://cloud.redhat.com/blog/delivering-high-accuracy-clock-synchronization-for-5g-networks-with-openshift-part-2](https://cloud.redhat.com/blog/delivering-high-accuracy-clock-synchronization-for-5g-networks-with-openshift-part-2) \

