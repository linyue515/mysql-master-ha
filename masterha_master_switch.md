
# masterha\_master\_switch #
[masterha\_manager](masterha_manager.md) is a program that both monitors and does failover the master. On the other hand, masterha\_master\_switch program does not monitor master. masterha\_master\_switch can be used for master failover, and also can be used for online master switch.

## Manual Failover ##
> Sometimes you might want to do failover manually. masterha\_master\_switch command can be used to run manual failover. The below is an example.

```
  $ masterha_master_switch --master_state=dead --conf=/etc/app1.cnf --dead_master_host=host1
```

> While [masterha\_manager](masterha_manager.md) command monitors master and does failover automatically, masterha\_master\_switch is intended to be used when you want to do manual failover.
> masterha\_master\_switch takes below arguments.

  * --master\_state=dead
> This a mandatory parameter. --master\_state takes either "dead" or "alive". If "alive" is set, masterha\_master\_switch starts online master switch operation. In that case, original master must be alive.

  * --dead\_master\_host=(hostname)
> Dead master's hostname. This is also a mandatory parameter. --dead\_master\_ip and --dead\_master\_port can also be optionally set. If these parameters are not set, --dead\_master\_ip will be the result of gethostbyname(dead\_master\_host), and --dead\_master\_port will be 3306.

  * --new\_master\_host=(hostname)
> New master's hostname. This parameter is optional. This parameter is useful when you want to explicitly set new master host (this means you do not want to let MHA decide the new master automatically). If new\_master\_host is not set, MHA determines new master based on the same rule as automated master failover (checking [candidate\_master parameter](Parameters#candidate_master.md), etc). If new\_master\_host is set, MHA determines the host as a new master. If the host can not be new master (i.e. log-bin is not enabled), MHA aborts.

  * --interactive=(0|1)
> If you want to do non-interactive failover described below, set --interactive=0. By default, it's 1 (interactive).

  * --ssh\_reachable=(0|1|2)
> Specifying whether the master is reachable via SSH or not. Set 0 if not reachable, set 1 if reachable, set 2 if unknown. Default is 2. If 2 is set, the command internally checks the master is reachable via SSH or not, and update the internal SSH status with 0 or 1.
> If the master is reachable via SSH, and if [master\_ip\_failover\_script](Parameters#master_ip_failover_script.md) or [shutdown\_script](Parameters#shutdown_script.md) is set, the command passes "--command=stopssh". If not, masterha\_master\_switch passes "--command=stop". In addition, if the crashed master is reachable via SSH, the failover script tries to copy unsent binary logs from the crashed master.

  * --skip\_change\_master
> By setting this argument on master failover, MHA exits after applying differential relay logs, and skipping CHANGE MASTER and START SLAVE. So slaves do not point to a new master. This may help if you want to manually double check whether slave recovery succeeded or not.

  * --skip\_disable\_read\_only
> By passing this parameter, MHA skips executing SET GLOBAL read\_only=0 on the new master. This may be useful when you want to manually disable .

  * --last\_failover\_minute=(minutes)
> Same as [masterha\_manager](masterha_manager.md).

  * --ignore\_last\_failover
> Same as [masterha\_manager](masterha_manager.md).

  * --wait\_on\_failover\_error=(seconds)
> Same as [masterha\_manager](masterha_manager.md).
> Note that this parameter applies to automated/non-interactive failover only, and this does not apply to interactive failover. That is, if --interactive=0 is not set, wait\_on\_failover\_error is simply ignored and does not sleep on errors.

  * --remove\_dead\_master\_conf
> Same as [masterha\_manager](masterha_manager.md).


> masterha\_master\_switch runs interactive failover procedures by default. You need to type "yes" from keyboard as below.

```
...
Starting master switch from host1(192.168.0.1:3306) to host2(192.168.0.2:3306)? (yes/NO): yes
...
```

> New master is determined by the same rule as automated failover, if --new\_master\_host is not set. When you run manual failover, you have an option to set new master explicitly. The below is an example.

```
Starting master switch from host1(192.168.0.1:3306) to host2(192.168.0.2:3306)? (yes/NO): no
Continue? (yes/NO): yes
Enter new master host name: host5
Master switch to gd1305(10.17.1.238:3306). OK? (yes/NO): yes
...
```

> In this case, host5 will be new master, as long as binary logging is enabled and major version is not higher than other slaves.
> The below command has the same effect as the above.

```
  $ masterha_master_switch --master_state=dead --conf=/etc/app1.cnf --dead_master_host=host1 --new_master_host=host5
```


  * --wait\_until\_gtid\_in\_sync(0|1)
> This option is available since 0.56. When doing GTID based failover, MHA waits until slaves to catch up the new master's GTID if setting wait\_until\_gtid\_in\_sync=1. If setting 0, MHA doesn't wait slaves to catch up. Default is 1.

  * --skip\_change\_master
> This option is available since 0.56. If this option is set, MHA skips executing CHANGE MASTER.

  * --skip\_disable\_read\_only
> This option is available since 0.56. If this option is set, MHA skips executing SET GLOBAL read\_only=0 on the new master.

  * --ignore\_binlog\_server\_error
> This option is available since 0.56. If this option is set, MHA ignores any error from binlog servers during failover.




## Non-Interactive Failover ##
> If you set "--interactive=0" in masterha\_master\_switch, it executes failover automatically (non-interactive).

```
  $ masterha_master_switch --master_state=dead --conf=/etc/conf/masterha/app1.cnf --dead_master_host=host1 --new_master_host=host2 --interactive=0
```

> This is actually the same what masterha\_manager runs internally. This non-interactive failover is useful if you have already verified that the master is dead, but you want to do failover as quickly as possible.
> Non-interactive failover is also useful if you use other existing master monitoring software and want to invoke non-interactive failover command from the software. Typical example is invoking masterha\_master\_switch from clustering software like Pacemaker.


## Scheduled(Online) Master Switch ##
> Sometimes you might want to do scheduled master switch, even though the current master is running. Typical example is replacing a partially broken hardware or upgrading the master server. You can not replace a RAID controller or increase memory without stopping the server. In such cases, you need to allocate a scheduled maintenance time, and you have to migrate the master to a different server.

> masterha\_master\_switch command can be used to run scheduled master switch.

```
  $ masterha_master_switch --master_state=alive --conf=/etc/app1.cnf --new_master_host=host2
```

> --master\_state=alive must be set.
> Program flows for the scheduled master switch is slightly different from the master failover. For example, you do not need to power off the master server, but you need to make sure that write queries are not executed on the master.
> By setting [master\_ip\_online\_change\_script](Parameters#master_ip_online_change_script.md), you can control how to disallow write traffics on the current master (i.e. dropping writable users, setting read\_only=1, etc) before executing FLUSH TABLES WITH READ LOCK, and how to allow write traffics on the new master.

> Online master switch starts only when all of the following conditions are met.

  * IO threads on all slaves are running
  * SQL threads on all slaves are running
  * Seconds\_Behind\_Master on all slaves are less or equal than --running\_updates\_limit seconds
  * On master, none of update queries take more than --running\_updates\_limit seconds in the show processlist output

> The reasons of these restrictions are for safety reasons, and to switch to the new master as quickly as possible.
> masterha\_master\_switch takes below arguments when switching master online.

  * --new\_master\_host=(hostname)
> New master's hostname.

  * --orig\_master\_is\_new\_slave
> After master switch completes, the previous master will run as a slave of the new master. By default, it's disabled (the previous master will not join new replication environments).
> If you use this option, you need to set [repl\_password](Parameters#repl_password.md) parameter in the config file because current master does not know the replication password for the new master.

  * --running\_updates\_limit=(seconds)
> If the current master executes write queries that take more than this parameter, or any of the MySQL slaves behind master more than this parameter, master switch aborts. By default, it's 1 (1 second).

  * --remove\_orig\_master\_conf
> When this option is set, if master switch succeeds correctly, MHA Manager automatically removes the section of the dead master from the configuration file. By default, the configuration file is not modified at all.

  * --skip\_lock\_all\_tables
> When doing master switch, MHA runs FLUSH TABLES WITH READ LOCK on a orig master to make sure updates are really stopped. But FLUSH TABLES WITH READ LOCK is very expensive and if you can make sure that no updates are coming to the orig master (by killing all clients at master\_ip\_online\_change\_script etc), you may want to avoid to lock tables by using this argument.