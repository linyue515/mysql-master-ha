

## MHA Node is not installed on one of MySQL servers ##
```
...
Sat Jul 2 13:24:25 2011 - [info] Checking MHA Node version..
Sat Jul 2 13:24:25 2011 - [error][/usr/lib/perl5/site_perl/5.8.5/MHA/ManagerUtil.pm, ln114] Got error when getting node version. Error:
Sat Jul 2 13:24:25 2011 - [error][/usr/lib/perl5/site_perl/5.8.5/MHA/ManagerUtil.pm, ln115]
bash: apply_diff_relay_logs: command not found
Sat Jul 2 13:24:25 2011 - [error][/usr/lib/perl5/site_perl/5.8.5/MHA/ManagerUtil.pm, ln130] node version on host2 not found! Maybe MHA Node package is not installed?
 at /usr/lib/perl5/site_perl/5.8.5/MHA/MasterMonitor.pm line 276
Sat Jul 2 13:24:25 2011 - [error][/usr/lib/perl5/site_perl/5.8.5/MHA/MasterMonitor.pm, ln316] Error happend while checking configurations. Died at /usr/lib/perl5/site_perl/5.8.5/MHA/ManagerUtil.pm line 131.
Sat Jul 2 13:24:25 2011 - [error][/usr/lib/perl5/site_perl/5.8.5/MHA/MasterMonitor.pm, ln397] Error happened while monitoring servers.
Sat Jul 2 13:24:25 2011 - [info] Got exit code 1 (Not master dead).
Died at /usr/bin/masterha_manager line 59.
```

> In this case, MHA Node package is not installed on host2. Download the package and install it.


## Could not find master's binary log ##

```
Sat Jul 3 20:03:40 2011 - [info] Checking MHA Node version..
Sat Jul 3 20:03:41 2011 - [info]  Version check ok.
Sat Jul 3 20:03:41 2011 - [info] Checking SSH publickey authentication and checking recovery script configurations on the current master..
Sat Jul 3 20:03:41 2011 - [info]   Executing command: save_binary_logs --command=test --start_file=binlog.000002 --start_pos=4 --binlog_dir=/var/lib/mysql,/var/log/mysql --output_file=/var/tmp/save_binary_logs_test --manager_version=0.50
Sat Jul 3 20:03:41 2011 - [info]   Connecting to root@hostx(192.168.0.1)..
Failed to save binary log: Binlog not found from /var/lib/mysql,/var/log/mysql!
 at /usr/bin/save_binary_logs line 95
        eval {...} called at /usr/bin/save_binary_logs line 59
        main::main() called at /usr/bin/save_binary_logs line 55
Sat Jul 3 20:03:41 2011 - [error][/usr/lib/perl5/site_perl/5.8.8/MHA/MasterMonitor.pm, ln94] Master setting check failed!
Sat Jul 3 20:03:41 2011 - [error][/usr/lib/perl5/site_perl/5.8.8/MHA/MasterMonitor.pm, ln296] Master configuration failed.
Sat Jul 3 20:03:41 2011 - [error][/usr/lib/perl5/site_perl/5.8.8/MHA/MasterMonitor.pm, ln316] Error happend on checking configurations.  at /usr/bin/masterha_manager line 50
```

In this example, master hostx stored binary logs on /data/mysql (by setting log-bin in my.cnf). MHA needs to know where binary logs are written on the master in order to save binary logs on failover. This can be set in [master\_binlog\_dir](Parameters#master_binlog_dir.md) parameter as below.

```
[server default]
master_binlog_dir=/data/mysql
```


## Read privilege on binary/relay logs are not granted ##

```
...
Sat Jul 2 13:27:21 2011 - [info] Checking SSH publickey authentication and checking recovery script configurations on the current master..
Sat Jul 2 13:27:21 2011 - [info]   Executing command: save_binary_logs --command=test --start_file=mysqld-bin.000001 --start_pos=4 --binlog_dir=/var/lib/mysql --output_file=/var/log/masterha/save_binary_logs_test --manager_version=0.50
Sat Jul 2 13:27:21 2011 - [info]   Connecting to app@host1(host1)..
Failed to save binary log: Permission denied:/var/lib/mysql/mysqld-bin.000001
 at /usr/bin/save_binary_logs line 96
Sat Jul 2 13:27:21 2011 - [error][/usr/lib/perl5/site_perl/5.8.5/MHA/MasterMonitor.pm, ln94] Master setting check failed!
Sat Jul 2 13:27:21 2011 - [error][/usr/lib/perl5/site_perl/5.8.5/MHA/MasterMonitor.pm, ln296] Master configuration failed.
Sat Jul 2 13:27:21 2011 - [error][/usr/lib/perl5/site_perl/5.8.5/MHA/MasterMonitor.pm, ln316] Error happend while checking configurations.  at /usr/bin/masterha_manager line 50
Sat Jul 2 13:27:21 2011 - [error][/usr/lib/perl5/site_perl/5.8.5/MHA/MasterMonitor.pm, ln397] Error happened while monitoring servers.
Sat Jul 2 13:27:21 2011 - [info] Got exit code 1 (Not master dead).
Died at /usr/bin/masterha_manager line 59.
```

> In this case, ssh\_user was app, but app user on host1 does not have read privilege on the binary log /var/lib/mysql/mysqld-bin.000001. To fix the problem, grant read privileges on binary logs to app user, or use stronger users such as root for SSH.

## Using multi-master replication (not supported) ##

```
...
Sat Jul 2 13:23:14 2011 - [error][/usr/lib/perl5/site_perl/5.8.5/MHA/ServerManager.pm, ln522] FATAL: Replication configuration error. All slaves should replicate from the same master.
Sat Jul 2 13:23:14 2011 - [error][/usr/lib/perl5/site_perl/5.8.5/MHA/ServerManager.pm, ln1066] MySQL master is not correctly configured. Check master/slave settings
Sat Jul 2 13:23:14 2011 - [error][/usr/lib/perl5/site_perl/5.8.5/MHA/MasterMonitor.pm, ln316] Error happend while checking configurations.  at /usr/lib/perl5/site_perl/5.8.5/MHA/MasterMonitor.pm line 243
Sat Jul 2 13:23:14 2011 - [error][/usr/lib/perl5/site_perl/5.8.5/MHA/MasterMonitor.pm, ln397] Error happened while monitoring servers.
Sat Jul 2 13:23:14 2011 - [info] Got exit code 1 (Not master dead).
Died at /usr/bin/masterha_manager line 59.
```

> The most likely cause of this error is that you are using multi-master replication. Currently MHA does not support multi-master replication. See [Requirements#Single\_master\_and\_multiple\_slaves](Requirements#Single_master_and_multiple_slaves.md) for details.