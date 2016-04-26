## Stopping Manager by masterha\_stop ##

> You can stop MHA Manager by masterha\_stop command.

```
  manager_host$ masterha_stop --conf=/etc/app1.cnf
  Stopped app1 successfully.
```

> masterha\_stop does NOT stop MySQL servers, just stopping monitoring.

> If it doesn't stop (i.e. hangs), add "--abort" argument. Then SIGKILL(-9) is sent to the process and all it's child processes.

> When the current manager status is FAILOVER\_RUNNING (failover operations are running), the script exits without stopping the manager process (--abort is also ignored). This is for safety reasons. Stopping failover in the middle will result in inconsistent replication settings and should be avoided.

> If you kill manager process manually (sending kill from shell), the process stops, but please make sure that it's not in the failover phase.
