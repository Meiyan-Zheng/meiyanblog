---
layout: post
title: How to check metallb status?
---

## Environment
- RHOCP 4.12
- Metallb operator

## Commands
```
export pod=<speaker podname>
$ oc exec -n metallb-system $pod -c frr -- vtysh -c "show bgp neighbor"
$ oc exec -n metallb-system $pod -c frr -- vtysh -c "show bgp summary"
$ oc exec -n metallb-system $pod -c frr -- vtysh -c "show bgp neighbor"
```
