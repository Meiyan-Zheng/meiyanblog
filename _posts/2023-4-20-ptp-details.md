---
layout: post
title: More details about PTP on OpenShift - WIP
---

## A high-level synchronization configuration using OpenShift

<img width="791" alt="Screenshot 2023-04-20 at 15 22 54" src="https://user-images.githubusercontent.com/30589773/233291005-f040624e-a267-4baa-bfb4-2203bdd64288.png">

In this diagram, we have different type of PTP clocks. 
* Grandmaster (GM) clock: This is the primary reference time clock (PRTC) for the entire PTP network. It usually synchronizes its clock from an external Global Navigation Satellite System (GNSS) source.
* Boundary clock (BC): This intermediate device has multiple PTP-capable network connections to synchronize one network segment to another accurately. It synchronizes its clock to a master and serves as a time source for ordinary clocks.
* Ordinary clock (OC): By contrast to boundary clocks, this device only has a single PTP-capable network connection. Its main function is to synchronize its clock to a master, and in the event of losing its master, it may tolerate a loss of sync source for some period of time.


## Reference Links:
[0] [https://cloud.redhat.com/blog/delivering-high-precision-clock-synchronization-for-low-latency-5g-networks-with-openshift-part-1](https://cloud.redhat.com/blog/delivering-high-precision-clock-synchronization-for-low-latency-5g-networks-with-openshift-part-1) \
[1] [https://cloud.redhat.com/blog/delivering-high-accuracy-clock-synchronization-for-5g-networks-with-openshift-part-2](https://cloud.redhat.com/blog/delivering-high-accuracy-clock-synchronization-for-5g-networks-with-openshift-part-2) \
[2] [https://www.techplayon.com/o-ran-fronthaul-transport-synchronization-configurations/](https://www.techplayon.com/o-ran-fronthaul-transport-synchronization-configurations/)
