---
layout: post
title: How to decode audit log?
---

I want to know which command were used from proctitle: 
```
# ausearch -k monitor-hosts
time->Tue Nov 28 11:45:57 2023
type=PROCTITLE msg=audit(1701143157.845:524997): proctitle=617564697463746C002D77002F7661722F6C6F672F706163656D616B65722F62756E646C657300002D7000776172002D6B006D6F6E69746F722D686F737473
type=SYSCALL msg=audit(1701143157.845:524997): arch=c000003e syscall=44 success=yes exit=1096 a0=4 a1=7ffe9d99d180 a2=448 a3=0 items=0 ppid=537026 pid=952862 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=245 comm="auditctl" exe="/usr/sbin/auditctl" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key=(null)
type=CONFIG_CHANGE msg=audit(1701143157.845:524997): auid=1000 ses=245 subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 op=add_rule key="monitor-hosts" list=4 res=1
time->Tue Nov 28 11:46:30 2023
type=PROCTITLE msg=audit(1701143190.320:525047): proctitle=63686D6F64006F2B72002F7661722F6C6F672F706163656D616B65722F62756E646C65732F7261626269746D712D62756E646C652D322F
type=PATH msg=audit(1701143190.320:525047): item=0 name="/var/log/pacemaker/bundles/rabbitmq-bundle-2/" inode=222367435 dev=fc:02 mode=040751 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:cluster_var_log_t:s0 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1701143190.320:525047): cwd="/root"
type=SYSCALL msg=audit(1701143190.320:525047): arch=c000003e syscall=268 success=yes exit=0 a0=ffffff9c a1=560747612670 a2=1ed a3=fffff3ff items=1 ppid=537026 pid=956001 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=245 comm="chmod" exe="/usr/bin/chmod" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="monitor-hosts"
```

Decode proctitle with xxd command:
```
$ echo 63686D6F64006F2B72002F7661722F6C6F672F706163656D616B65722F62756E646C65732F7261626269746D712D62756E646C652D322F | xxd -r -p; echo
chmodo+r/var/log/pacemaker/bundles/rabbitmq-bundle-2/
```

Now we know the file permission change was caused by command chmod +r /var/log/pacemaker/bundles/rabbitmq-bundle-2/
