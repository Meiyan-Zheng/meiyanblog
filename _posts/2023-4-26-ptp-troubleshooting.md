---
layout: post
title: PTP troubleshooting on openshift 
---

### Check NIC driver & firmware version
Login to openshift nodes with ptp pod running on: 
```
sh-4.4# ethtool -i ens1f0
driver: ice
version: 1.11.16 <------- driver version
firmware-version: 4.20 0x8001778b 1.3346.0 <--------- firmware version
expansion-rom-version: 
bus-info: 0000:10:00.0
supports-statistics: yes
supports-test: yes
supports-eeprom-access: yes
supports-register-dump: yes
supports-priv-flags: yes

```

### Check interfaces connected to gnss device
Login to openshift nodes with ptp pod running on: 
```
sh-4.4# find /sys/devices/ -name gnss
/sys/devices/pci0000:0f/0000:0f:02.0/0000:10:00.0/gnss
/sys/devices/pci0000:4a/0000:4a:02.0/0000:4b:00.0/gnss
sh-4.4# ls /sys/devices/pci0000:0f/0000:0f:02.0/0000:10:00.0/gnss
gnss0
sh-4.4# ls /sys/devices/pci0000:4a/0000:4a:02.0/0000:4b:00.0/gnss
gnss1
sh-4.4# ls /sys/devices/pci0000:0f/0000:0f:02.0/0000:10:00.0/net
ens1f0
sh-4.4# ls /sys/devices/pci0000:4a/0000:4a:02.0/0000:4b:00.0/net
ens2f0
```

In this example, NIC which having interface `ens1f0` is connected to device `/dev/gnss0`, and NIC which having interface `ens2f0` is connected to device `/dev/gnss1`

### Check if gnss device is working fine
Login to openshift nodes with ptp pod running on: 
```
sh-4.4# cat /dev/gnss0
$GNRMC,,V,,,,,,,,,,N,V*37
$GNGGA,,,,,,0,00,99.99,,,,,,*56
$GNRMC,,V,,,,,,,,,,N,V*37
$GNGGA,,,,,,0,00,99.99,,,,,,*56

sh-4.4# cat /dev/gnss1
$GNRMC,070031.00,A,4233.01542,N,07112.87799,W,0.005,,260423,,,A,V*0E
$GNGGA,070031.00,4233.01542,N,07112.87799,W,1,09,0.95,60.0,M,-33.0,M,,*42
$GNRMC,070032.00,A,4233.01542,N,07112.87799,W,0.005,,260423,,,A,V*0D
$GNGGA,070032.00,4233.01542,N,07112.87799,W,1,09,0.95,59.9,M,-33.0,M,,*42
```

In this example, device `gnss0` is not getting data. If it's getting data from GPS, the output should be like device `gnss1`.

### Check ptp configuration
```
$ oc get ptpConfig -n openshift-ptp
NAME           AGE
grandmaster    26d
grandmaster2   26d

For more detailed configuration, we can use `-o yaml`:
$ oc get ptpConfig -n openshift-ptp grandmaster -o yaml
...
```

And those configurations also can be find in `linuxptp-daemon` pod:
```
$ oc get pod -n openshift-ptp
NAME                            READY   STATUS    RESTARTS   AGE
linuxptp-daemon-2gkhd           2/2     Running   8          26d <---- linuxptp-daemon pod
ptp-operator-6cf44c55df-b9m8v   1/1     Running   4          26d

$ oc rsh -n openshift-ptp -c linuxptp-daemon-container linuxptp-daemon-2gkhd
sh-4.4# ls -l /var/run/
total 16
-rw-r--r--. 1 root root  105 Apr 25 10:34 phc2sys.0.config
-rw-r--r--. 1 root root 1786 Apr 25 10:34 ptp4l.0.config
srw-rw----. 1 root root    0 Apr 25 10:34 ptp4l.0.socket
-rw-r--r--. 1 root root 1787 Apr 25 10:34 ptp4l.1.config
srw-rw----. 1 root root    0 Apr 25 10:34 ptp4l.1.socket
srw-rw----. 1 root root    0 Apr 25 10:34 ptp4lro
drwxr-xr-x. 4 root root   80 Apr 17 14:27 secrets
-rw-r--r--. 1 root root  419 Apr 25 10:34 ts2phc.0.config
```

### Check ptp logs for each daemon
```
$ oc logs -n openshift-ptp -c linuxptp-daemon-container linuxptp-daemon-2gkhd | grep phc2sys
...
phc2sys[739493.664]: [ptp4l.0.config] CLOCK_REALTIME phc offset         3 s2 freq  -23571 delay    511
phc2sys[739493.727]: [ptp4l.0.config] CLOCK_REALTIME phc offset       -13 s2 freq  -23586 delay    519
phc2sys[739493.789]: [ptp4l.0.config] CLOCK_REALTIME phc offset        10 s2 freq  -23567 delay    511
phc2sys[739493.852]: [ptp4l.0.config] CLOCK_REALTIME phc offset        -6 s2 freq  -23580 delay    511
phc2sys[739493.914]: [ptp4l.0.config] CLOCK_REALTIME phc offset         4 s2 freq  -23572 delay    511
phc2sys[739493.977]: [ptp4l.0.config] CLOCK_REALTIME phc offset         4 s2 freq  -23571 delay    510
```

`offset`: The time differences between NIC clock and system clock. A low level (usually < 100 nanoseconds) means the signal quality is good. Note that, right after the PTP daemon is started, there may be a big offset, which should be corrected after some minutes.
\
`delay`: \
`freq`: \

```
$ oc logs -n openshift-ptp -c linuxptp-daemon-container linuxptp-daemon-2gkhd | grep ts2phc
...
ts2phc[739737.773]: [ts2phc.0.config] nmea sentence: GNRMC,035315.00,A,4233.01536,N,07112.87851,W,0.008,,260423,,,A,V
ts2phc[739738.387]: [ts2phc.0.config] nmea delay: 63968727 ns
ts2phc[739738.387]: [ts2phc.0.config] ens1f0 extts index 0 at 1682481233.000000000 corr 0 src 1682481233.614465216 diff 0
ts2phc[739738.387]: [ts2phc.0.config] ens1f0 master offset          0 s2 freq      -0
ts2phc[739738.709]: [ts2phc.0.config] nmea delay: 63968727 ns
ts2phc[739738.709]: [ts2phc.0.config] ens2f0 extts index 0 at 1682481233.000000000 corr 0 src 1682481233.936055366 diff 0
ts2phc[739738.709]: [ts2phc.0.config] ens2f0 master offset          0 s2 freq      -0
ts2phc[739738.764]: [ts2phc.0.config] nmea sentence: GNRMC,035316.00,A,4233.01536,N,07112.87851,W,0.008,,260423,,,A,V
```

`master offset`: \

```
$ oc logs -n openshift-ptp -c linuxptp-daemon-container linuxptp-daemon-2gkhd | grep ptp4l 
...

```


