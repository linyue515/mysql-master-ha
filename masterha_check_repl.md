## Checking MySQL Replication Health by masterha\_check\_repl ##

> Here is an example.

```
  manager_host$ masterha_check_repl --conf=/etc/app1.cnf
  ...
  MySQL Replication Health is OK.
```

> masterha\_check\_repl connects all MySQL servers defined at the config file, then checking replication works correctly or not.
> This command does not conflict with masterha\_manager. You can run this command while masterha\_manager is working (monitoring the current master). This command is useful if you want to check replication settings regularly. After masterha\_manager monitors MySQL master, it only checks master server's availability. By running masterha\_check\_repl regularly and send alerts if it gets errors (i.e. SQL thread stops), you will be able to quickly fix replication problems.

> masterha\_check\_repl returns errors when detecting any replication failure. Replication failures include the followings.
  * All replication failures that MHA can't monitor/failover (replication filtering rule is different, SQL thread is stopped with errors, if you have two or more masters, etc)
  * IO thread is stopped
  * SQL thread is stopped normally
  * Replication delays more than N seconds


> masterha\_check\_repl takes below arguments, in addition to --conf.

  * --seconds\_behind\_master=(seconds)
> masterha\_check\_repl checks Seconds\_Behind\_Master from all slaves and return errors if it exceeds threshold. By default, it's 30 (seconds).

  * --skip\_check\_ssh
> By default, masterha\_check\_repl executes as the same check scripts as masterha\_check\_ssh. If you are sure that SSH connection checks are not needed, use this option.