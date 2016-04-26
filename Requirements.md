
# Requirements and Limitations #

To make MHA really work, you need to verify the following settings. Most of the settings are automatically checked at starting [masterha\_manager](masterha_manager.md) or [masterha\_check\_repl](masterha_check_repl.md).

# SSH public key authentication #
MHA Manager internally connects to MySQL servers via SSH. MHA Node on the latest slave also internally sends relay log files to other slaves via SSH(scp).
To make these procedures automated, it is generally recommended to enable SSH public key authentication without pass-phrase.
You can use masterha\_check\_ssh command included in MHA Manager to check whether SSH connections work properly.

```
  # masterha_check_ssh --conf=/etc/app1.cnf
  
  Sat May 14 14:42:19 2011 - [warn] Global configuration file /etc/masterha_default.cnf not found. Skipping.
  Sat May 14 14:42:19 2011 - [info] Reading application default configurations from /etc/app1.cnf..
  Sat May 14 14:42:19 2011 - [info] Reading server configurations from /etc/app1.cnf..
  Sat May 14 14:42:19 2011 - [info] Starting SSH connection tests..
  Sat May 14 14:42:19 2011 - [debug]  Connecting via SSH from root@host1(192.168.0.1) to root@host2(192.168.0.2)..
  Sat May 14 14:42:20 2011 - [debug]   ok.
  Sat May 14 14:42:20 2011 - [debug]  Connecting via SSH from root@host1(192.168.0.1) to root@host3(192.168.0.3)..
  Sat May 14 14:42:20 2011 - [debug]   ok.
  Sat May 14 14:42:21 2011 - [debug]  Connecting via SSH from root@host2(192.168.0.2) to root@host1(192.168.0.1)..
  Sat May 14 14:42:21 2011 - [debug]   ok.
  Sat May 14 14:42:21 2011 - [debug]  Connecting via SSH from root@host2(192.168.0.2) to root@host3(192.168.0.3)..
  Sat May 14 14:42:21 2011 - [debug]   ok.
  Sat May 14 14:42:22 2011 - [debug]  Connecting via SSH from root@host3(192.168.0.3) to root@host1(192.168.0.1)..
  Sat May 14 14:42:22 2011 - [debug]   ok.
  Sat May 14 14:42:22 2011 - [debug]  Connecting via SSH from root@host3(192.168.0.3) to root@host2(192.168.0.2)..
  Sat May 14 14:42:22 2011 - [debug]   ok.
  Sat May 14 14:42:22 2011 - [info] All SSH connection tests passed successfully.
```

When you use masterha\_secondary\_check script to check additional network connectivity, you also need to verify that SSH public key accesses from MHA Manager to remote hosts are available.

MHA Manager internally executes recovery in parallel. When generating differential relay log files, MHA connects to the latest slave via SSH in parallel and generates and sends differential relay logs to non-latest slaves. If you have tens of slaves or consolidate tens of MySQL instances within single OS, sshd might reject SSH connection requests,  which will cause slave recovery errors. To avoid this issue, raise MaxStartups parameter in /etc/ssh/sshd\_config (default 10) and restart sshd.

# Operating Systems #
MHA is tested on Linux only.

# Single writable master and multiple slaves or read-only masters #
MHA fixes consistency problems between slaves in case of the mater crash. Additionally MHA tries to save unsent binary log events from the crashed master
and apply events to all slaves. If you have only one slave, you don't need to care about consistency issues between slaves.
Even if you use only one slave, MHA is still beneficial for saving events from the crashed master as long as the crashed master is reachable via SSH, but using semi-synchronous replication can also solve this issue.

Starting from MHA Manager version 0.52, multi-master replication configurations are supported. Here are some notes to make MHA work with multi-master.

  * Only one primary master (writable) is allowed. MySQL global variable "read-only=1" must be set on other MySQL masters.
  * By default, all managed servers (defined in a MHA configuration file) should be in two-tier replication channel. See below for details.

Prior to MHA Manager version 0.52, MHA checks all managed hosts and starts monitoring/failover only if all slaves replicate from the same master. That is, MHA did NOT support Multi-Master configuration. In general, as long as MHA manages automated failover,
the reason for using multi-master configuration is limited, such as online schema changes. If you want to use
multi-master configuration tentatively (i.e. just for schema change), just stop MHA and
configure multi-master during operations. After the operation completes, make single master and
multiple slave configuratoins again, then restart MHA.


# For managing three or more tier replication environment.. #
MHA does not support three or more tier replication structure by default(i.e. Master1 -> Master2 -> Slave3).
MHA does failover and recover only slaves that directly replicate from the current primary master. MHA can manage master1 and master2 and
promote master2 when master1 crashes, but MHA can not monitor and recover slave3 because slave3 replicates from different master(master2). To make MHA work with such structures, either configure as below.

  * Just set master and second-tier hosts (master1 and master2) in this case in the MHA configuration file
  * Use ["multi\_tier\_slave=1"](Parameters#multi_tier_slave.md) parameter and set all hosts in the MHA configuration file

In both cases, MHA just manage the primary master and second tier slaves. This is still very helpful because automated failover from master1 to master2 is possible in case of the primary master (master1) crash, and third tier slaves can still work. See [UseCases](UseCases#Three_tier_replication.md) page for details.


# MySQL version 5.0 or later #
MHA supports MySQL version 5.0 or higher. MHA does NOT support MySQL 4.1 or earlier versions. This is because
binary log format changed from MySQL 5.0 (called binlog v4 format). When MHA parses binary logs to identify target relay log position, binlog format has to be v4 so MySQL 4.1 does not work here.
mysqlbinlog in MySQL 5.0 or higher also requires that binlog format is v4.
It is also important that earlier versions of MySQL has some serious issues around MySQL replication handling, so it is highly recommended using higher version.
Especially if you use before MySQL 5.0.60, consider upgrading it.

# Use mysqlbinlog 5.1+ for MySQL 5.1+ #
MHA uses mysqlbinlog for applying binlog events to target slaves. If the master uses row based format,
row based events are written in the binary log. Such binary logs can be parsed by mysqlbinlog in MySQL 5.1 or higher only
because mysqlbinlog in MySQL 5.0 does not support row based format.
mysqlbinlog (and mysql) version can be checked as below.

```
  [app@slave_host1]$  mysqlbinlog --version
  mysqlbinlog Ver 3.3 for unknown-linux-gnu at x86_64
```

mysqlbinlog version should be 3.3 or higher if it's included in MySQL 5.1.
MHA internally checks both MySQL and mysqlbinlog version on all slaves. If mysqlbinlog version is 3.2 or earlier
and if MySQL version is 5.1 or higher, MHA stops with errors before starting monitoring.

# log-bin must be enabled on candidate masters #
If current slaves do not set log-bin, obviously they can not be the new master.
MHA Manager internally checks log-bin settings, and do not promote them to new master. If none of the current slaves set log-bin, MHA Manager does not proceed failover.

# Binary log and relay log filtering rules must be same on all servers #

Replication filtering rules (binlog-do-db, replicate-ignore-db, etc) must be the same on all MySQL servers. MHA checks filtering rules at startup, and doesn't start monitoring or failover if filtering rules are not same each other.

# Replication user must exist on candidate masters #
After master failover completes, all other slaves execute CHANGE MASTER TO statement.
To start replication, a replication user
(with REPLICATEION SLAVE privilege) must exist on the new master.

# Preserve relay logs and purge regularly #
By default, relay logs on slave servers are automatically removed if SQL threads have finished executing them.
But such relay logs might still be needed for recovering other slaves.
So you need to disable automatic relay log purge, and periodically purges old relay logs.
But you need to care about replication delay issues when manually purging relay logs. On ext3 filesystem, removing large files takes significant time, which will cause serious replication delay. To avoid replication delay, tentatively creating hard links of the relay logs helps.
See the [two](http://www.slideshare.net/matsunobu/automated-master-failover/26) [slides](http://www.slideshare.net/matsunobu/automated-master-failover/27) for details.

## purge\_relay\_logs script ##

MHA Node has a command line tool “purge\_relay\_logs” to do that.
purge\_relay\_logs does all things described in the above slides. That is, it creates hard links, executing "SET GLOBAL relay\_log\_purge=1", waiting a few seconds
so that SQL thread can switch to the new relay log (as long as it behinds significantly), and executing "SET GLOBAL relay\_log\_purge=0".

purge\_relay\_logs takes below arguments. purge\_relay\_logs internally connects to MySQL slave server so MySQL authentication arguments are needed.

  * --user
> MySQL username. By default, it's root.

  * --password
> MySQL password for --user. By default, it's empty.

  * --host
> MySQL hostname. By default, it's 127.0.0.1. --host must be the same server where you run purge\_relay\_logs. For example, if you pass --host=host2 on host1, purge\_relay\_logs aborts. If you have virtual hostname vhost1 on host1, passing --host=vhost1 on host1 is valid and purge\_relay\_logs does not abort.

  * --port
> MySQL port number. By default, it's 3306.

  * --workdir
> Tentative directory where hard linked relay logs are created and removed. After executing the script successfully, hard-linked relay log files are deleted. By default, it's /var/tmp.
> If you put relay log files on a different OS partition from /var/tmp, you need to explicitly set this parameter. This is because creating hard links between different OS partitions are not supported (You'll get "Invalid cross-device link" error). In this case, set --workdir to a directory which resides on a same OS partition as relay log files.

  * --disable\_relay\_log\_purge
> By default, if relay\_log\_purge is ON(1) in MySQL, purge\_relay\_logs script exits without doing anything. So relay\_log\_purge is still ON(1). By setting --disable\_relay\_log\_purge, purge\_relay\_logs script does not exit and automatically sets relay\_log\_purge to 0. So after executing the script, relay\_log\_purge becomes OFF(0).



## Schedule to run purge\_relay\_logs script ##
pureg\_relay\_logs removes relay logs without blocking SQL threads.
Relay logs need to be purged regularly (i.e. once per day, once per 6 hours, etc), so purge\_relay\_logs should be regularly invoked on each slave server from job schedulers.
For example, you can invoke purge\_relay\_logs from cron as below.

```
  [app@slave_host1]$ cat /etc/cron.d/purge_relay_logs
  # purge relay logs at 5am
  0 5 * * * app /usr/bin/purge_relay_logs --user=root --password=PASSWORD --disable_relay_log_purge >> /var/log/masterha/purge_relay_logs.log 2>&1
```

It is recommended invoking cron at different time between slaves. If all slaves invoke purge\_relay\_logs at the same time, none of the slave might not have necessary relay log events at crash.




# Do not use LOAD DATA INFILE with Statement Based Binary Logging #

When the master is crashed just after completing LOAD DATA INFILE with Statement Based Binary Logging, MHA might not be able to generate differential relay log events, if you use non-transactional storage engine or too old MySQL version (i.e. 5.0.45, etc). Using LOAD DATA INFILE with SBR has some known issues and it's deprecated from MySQL 5.1. LOAD DATA INFILE also causes significant replication delay, so there are not positive reasons to use it.

If you want to use LOAD DATA, SET sql\_log\_bin=0; LOAD DATA … ; SET sql\_log\_bin=1; is more recommended approach.