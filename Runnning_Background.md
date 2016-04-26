# Running MHA Manager in background #
> By default, masterha\_manager runs in foreground. You can run the masterha\_manager program in background as below.

```
  manager_host$ nohup masterha_manager --conf=/etc/app1.cnf < /dev/null > /var/log/masterha/app1/app1.log 2>&1 &
```

> Set nohup, and make sure that masterha\_manager does not read/write from/to STDIN, STDOUT and STDERR.


# Running MHA Manager from daemontools #
> Currently MHA Manager process does not run as a daemon. If failover completed successfully or the master process was killed by accident, the manager stops working.
> To run as a daemon, daemontool. or any external daemon program can be used. Here is an example to run from daemontools.

1. Install daemontools

```
  * For RedHat
  manager_host# yum install daemontools
```

2. Create run file under /service/masterha_(app\_name)/run_

```
  manager_host# mkdir /service/masterha_app1
  manager_host# cat /service/masterha_app1/run
  #!/bin/sh
  exec masterha_manager --conf=/etc/app1.cnf --wait_on_monitor_error=60 --wait_on_failover_error=60 >> /var/log/masterha/app1/app1.log 2>&1
  manager_host# chmod 755 /service/masterha_app1/run
```

> You can stop/restart monitoring by daemontool commands.

```
  ## stopping monitoring
  manager_host# svc -d /service/masterha_app1
  
  ## starting monitoring
  manager_host# svc -u /service/masterha_app1
```