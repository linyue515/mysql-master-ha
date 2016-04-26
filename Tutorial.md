
# Tutorial #

## Simple Failover ##

### Building normal replication environments ###
> MHA does not build replication environments by itself, so you need to setup replication by yourself. In other words, you can use MHA on existing environments.
> As an example, suppose there are four hosts: host1, host2, host3, host4. Master is currently running on host1, and two slaves are running on host2 and host3. So MySQL servers are running on host1, host2 and host3. Let's run MHA Manager on host4(manager\_host).


### Installing MHA Node on host1-host4 ###
> See [Installing MHA Node](Installation#Installing_MHA_Node.md)

### Installing MHA Manager on host4(manager\_host) ###
> See [Installing MHA Manager](Installation#Installing_MHA_Manager.md). MHA Manage depends on MHA Node package so you need to install both on the manager server.

### Creating a config file ###
> Next step is creating a configuration file on MHA Manager. Parameters include hostname of each MySQL server, MySQL username and password, MySQL replication username and password, working directory name, etc. The whole parameters are described at [Parameters](Parameters.md) page.


```
  manager_host$ cat /etc/app1.cnf
  
  [server default]
  # mysql user and password
  user=root
  password=mysqlpass
  ssh_user=root
  # working directory on the manager
  manager_workdir=/var/log/masterha/app1
  # working directory on MySQL servers
  remote_workdir=/var/log/masterha/app1
  
  [server1]
  hostname=host1
  
  [server2]
  hostname=host2
  
  [server3]
  hostname=host3
```

> Note that you do not specify that host1 is a current master. MHA automatically detects current master internally.


### Checking SSH connections ###
> MHA Manager internally invokes programs included in the MHA Node package via SSH. MHA Node programs also send differential relay log files to other non-latest slaves via SSH (scp). To make these procedures non-interactive, it is necessary to setup SSH public key authentication. MHA Manager provides a simple check program "masterha\_check\_ssh" to verify non-interactive SSH connections can be established each other.

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

> If masterha\_check\_ssh stops with errors or authentication requests, SSH configuration is not valid for MHA to work. You need to fix it and try again. Most possible cause is SSH public key authentication is not set properly.


### Checking Replication Configuration ###
> To make MHA work, all MySQL master and slaves defined in the config file must work correctly. MHA Manager provides a command [masterha\_check\_repl](masterha_check_repl.md) to quickly check replication health.

```
  manager_host$ masterha_check_repl --conf=/etc/app1.cnf
  ...
  MySQL Replication Health is OK.
```

> If you get any error here, check logs and fix issues. Current master must not be a slave, and all other slaves must replicate from the master. [TypicalErrors](TypicalErrors.md) page may help to fix setup errors.

### Starting Manager ###
> You configured MySQL replication, installed both MHA Node and MHA Manager, and configured SSH public key authentication. Next step is starting MHA Manager. MHA Manager can be started by [masterha\_manager](masterha_manager.md) command.

```
  manager_host$ masterha_manager --conf=/etc/app1.cnf
  ....
  Sat May 14 15:58:29 2011 - [info] Connecting to the master host1(192.168.0.1:3306) and sleeping until it doesn't respond..
```

> If all configurations are valid, [masterha\_manager](masterha_manager.md) checks MySQL master availability until the master dies. If [masterha\_manager](masterha_manager.md) stops with errors before monitoring the master, check error logs and fix configurations. All logs are printed to standard error(STDERR) by default, but this can be changed in ["manager\_log" configuration parameter](Parameters#manager_log.md). Typical errors are that MySQL replication configuration is not valid, ssh\_user does not have sufficient privileges (minimum requirements are read privileges for relay logs and write privileges on remote\_workdir).
> By default, [masterha\_manager](masterha_manager.md) runs in foreground. If you send SIGINT(sending Ctrl+C) to masterha\_manager, masterha\_manager stops from monitoring and exits.


### Checking manager status ###
> After MHA Manager monitors MySQL master, it doesn't print anything until master becomes unreachable or terminating manager itself. You may want to check whether MHA Manager really works fine or not. [masterha\_check\_status](masterha_check_status.md) command is useful to check current MHA Manager status. The below is an example.

```
  manager_host$ masterha_check_status --conf=/etc/app1.cnf
  app1 (pid:5057) is running(0:PING_OK), master:host1
```

> "app1" is an application name that MHA internally handles, which is a prefix name of the configuration file.

> If manager stops or configuration file is invalid, the following error will be returned.

```
  manager_host$ masterha_check_status --conf=/etc/app1.cnf
  app1 is stopped(1:NOT_RUNNING).
```


### Stopping manager ###
> You can stop MHA Manager by [masterha\_stop](masterha_stop.md) command.

```
  manager_host$ masterha_stop --conf=/etc/app1.cnf
  Stopped app1 successfully.
```

> If it doesn't stop (i.e. hangs), add "--abort" argument.

> Let's start [masterha\_manager](masterha_manager.md) again once you understand how to stop it.


### Testing master failover ###
> Now MHA Manager monitors MySQL master server availability. Next, let's test that master failover works correctly. To simulate this, you can simply kill mysqld on the master.

```
  host1$  killall -9 mysqld mysqld_safe
```

> On some distributions like Ubuntu, mysqld will be automatically restarted by angel process. If mysqld restarts very quickly (a few seconds), pings from MHA will succeed again before MHA starts failover. In such cases, failover does not start. If restarting mysqld takes long time (i.e. taking 2 minutes for InnoDB crash recovery), failover will start.

> If you have difficulties for testing killing mysqld or if you want to test Linux kernel side problem, invoking kernel panic is easy.

```
  host1#  echo c > /proc/sysrq-trigger
```

> Check logs on MHA manager, and verify that host2 becomes new mater, and host3 replicates from host2.

> When failover completes (or ends with errors), MHA Manager process stops. This is an expected behavior. If you want to run MHA Manager permanently, please read ["Running MHA Manager in background"](Runnning_Background.md) section.


## Next Steps ##
> Now we have covered basic tests for master failover. In practice, you may want to do many things like below.

  * Checking MySQL Master availability via two or more network routes
> See [secondary\_network\_script parameter](Parameters#secondary_check_script.md) for details.

  * Writing a master ip failover script
> In the above tutorial, new master's IP address (host2's IP) has changed from the dead master's IP address (host1's IP). So applications have to change master's IP address. This is not fun. To solve this issue, using [master\_ip\_failover\_script parameter](Parameters#master_ip_failover_script.md) helps. You can write any program to update master's IP address. For example, taking over virtual IP address, updating global catalog database, etc. A sample script is included in MHA Manager package (See (MHA Manager dir)/samples/scripts/master\_ip\_failover). Sample scripts are included in MHA Manager tarball and GitHub branch.

  * Running MHA Manager in background
> See [Runnning\_Background](Runnning_Background.md) page for details.

  * Power off MySQL Master so that split brain will never happen
> See [shutdown\_script parameter](Parameters#shutdown_script.md) and a sample script (MHA Manager dir)/samples/scripts/power\_manager included in MHA Manager package.

  * Sending e-mail when failover completes
> See [report\_script parameter](Parameters#report_script.md) and a sample script (MHA Manager dir)/samples/scripts/send\_report in MHA Manager package.

  * Failover error and manual failover tests
> You may want to test failure scenario. See the below example.

```
  terminal1# masterha_manager --conf=/etc/app1.cnf
  terminal2# mysql -hhost2 db1 -e "insert into t1 values (100, 100, 100)"
  terminal2# mysql -hhost1 db1 -e "insert into t1 values (100, 100, 100)"
  Check replication stops with error
  kill master
  Check master failover does not work.
  mysql_host2> set global slave_sql_skip_counter=1;
  mysql_host2> start slave;
  Check replication starts again.

  Test manual failover
  # remove error file
  # rm -f /var/log/masterha/mha_test50/mha_test50.failover.error
  # masterha_master_switch  --master_state=dead --conf=/path/to/conf --dead_master_host=host1
```