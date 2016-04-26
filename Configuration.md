

# Writing an application configuration file #

> To make MHA work, you have to create a configuration file and set parameters. Parameters include hostname of each MySQL server, MySQL username and password, working directory name, etc. The whole parameters are described at [Parameters](Parameters.md) page.

> The below is an example configuration file.

```
  manager_host$ cat /etc/app1.cnf
  
  [server default]
  # mysql user and password
  user=root
  password=mysqlpass
  # working directory on the manager
  manager_workdir=/var/log/masterha/app1
  # manager log file
  manager_log=/var/log/masterha/app1/app1.log
  # working directory on MySQL servers
  remote_workdir=/var/log/masterha/app1
  
  [server1]
  hostname=host1
  
  [server2]
  hostname=host2
  
  [server3]
  hostname=host3
```

> All parameters must follow "param=value" syntax. For example, the below parameter setting is incorrect.

```
  [server1]
  hostname=host1
  # incorrect: must be "no_master=1"
  no_master
```



> Application-scope parameters should be written in `[server default]` block.
> In `[serverN]` blocks, you should set local-scope parameters. hostname is mandatory local-scope parameter so has to be written here. Block name should start from "server". Internally server configurations are sorted by block name, and sorted order matters when MHA decides new master (See [FAQ](FAQ.md) for details).

# Writing a global configuration file #

> If you plan to manage two or more MySQL applications ((master, slaves) pairs) from a single manager server, creating a global configuration file makes it much easier to configure.
> Once you write parameters in the global configuration file, you do not need to set parameters for each application. If you create a file at /etc/masterha\_default.cnf, MHA Manager scripts automatically reads the file as a global configuration file.

> You can set application scope parameters in the global configuration file. For example, if MySQL administrative user and password are identical on all MySQL servers, you can set "user" and "password" here.


## Global Configuration Example: ##

> Global configuration file (/etc/masterha\_default.cnf)

```
  [server default]
  user=root
  password=rootpass
  ssh_user=root
  master_binlog_dir= /var/lib/mysql
  remote_workdir=/data/log/masterha
  secondary_check_script= masterha_secondary_check -s remote_host1 -s remote_host2
  ping_interval=3
  master_ip_failover_script=/script/masterha/master_ip_failover
  shutdown_script= /script/masterha/power_manager
  report_script= /script/masterha/send_master_failover_mail
```

> These parameters are applied to all applications monitored by MHA Manager running on the host.

> Application configuration file should be written separately. The below example is configuraing app1 (host1-4) and app2 (host11-14).

app1:

```
  manager_host$ cat /etc/app1.cnf

  [server default]
  manager_workdir=/var/log/masterha/app1
  manager_log=/var/log/masterha/app1/app1.log
  
  [server1]
  hostname=host1
  candidate_master=1
  
  [server2]
  hostname=host2
  candidate_master=1
  
  [server3]
  hostname=host3
  
  [server4]
  hostname=host4
  no_master=1
```

In the above case, MHA Manager generates working files (including status files) under /var/log/masterha/app1, and generates log file at /var/log/masterha/app1/app1.log. You need to set unique directory/file names when monitoring other applications.

app2:

```
  manager_host$ cat /etc/app2.cnf

  [server default]
  manager_workdir=/var/log/masterha/app2
  manager_log=/var/log/masterha/app2/app2.log
  
  [server1]
  hostname=host11
  candidate_master=1
  
  [server2]
  hostname=host12
  candidate_master=1
  
  [server3]
  hostname=host13
  
  [server4]
  hostname=host14
  no_master=1
```

> If you set same parameters on both global configuration file and application configuration file, the latter (application configuration) is used.


# Binlog server #

> Starting from MHA version 0.56, MHA supports new section `[binlogN]`. In binlog section, you can define [mysqlbinlog streaming servers](http://dev.mysql.com/doc/refman/5.6/en/mysqlbinlog-backup.html). When MHA does GTID based failover, MHA checks binlog servers, and if binlog servers are ahead of other slaves, MHA applies differential binlog events to the new master before recovery. When MHA does non-GTID based (traditional) failover, MHA ignores binlog servers.

> Below is an example configuration.

```
  manager_host$ cat /etc/app1.cnf
  
  [server default]
  # mysql user and password
  user=root
  password=mysqlpass
  # working directory on the manager
  manager_workdir=/var/log/masterha/app1
  # manager log file
  manager_log=/var/log/masterha/app1/app1.log
  # working directory on MySQL servers
  remote_workdir=/var/log/masterha/app1
  
  [server1]
  hostname=host1
  
  [server2]
  hostname=host2
  
  [server3]
  hostname=host3

  [binlog1]
  hostname=binlog_host1

  [binlog2]
  hostname=binlog_host2
```