

# Frequently Asked Questions #

# Which MySQL versions are supported? #

> MySQL 5.0 or higer versions are supported. MySQL 4.1 or lower versions are NOT supported.

> MHA also supports 5.6+ system table replication info (managing relay log pos within system table) transparently.

# Which operating systems are supported? #

> MHA supports all Linux distributions. MHA should work other UNIX platforms, but it has not been tested yet. MHA does NOT support Windows.

# Is MHA reliable? #
> Yes, as long as you are using two-tier (regular master-slaves) replication. In two-tier replication, master's binary log file and position are written in the slave's relay log (See [slides p.17-](http://www.slideshare.net/matsunobu/automated-master-failover/17) for details) so it works as global transaction id. So MHA can identify exact starting position by parsing relay logs, as long as slave servers are alive. By default, MHA does not start failover unless all of the slaves are alive, so when MHA works it should be reliable.

# Do you offer commercial support for MHA? #
> Yes, [SkySQL](http://www.skysql.com/en/index) provides commercial support for MHA.

# Is row based binlog format supported? #

> Yes. Any binary logging format including statement based and row based (and mixed) are supported.
> One known issue is that when master fails just after executing large LOAD DATA with statement based binary logging, recovery might fail. Using LOAD DATA with statement based binary logging is deprecated in MySQL so please do not use it.

# Does MHA work on cloud environments? #

> If you have a root account, yes, you can. MHA needs OS accounts on MySQL servers in order to read binary or relay logs, and generating working files. Reading binary/relay logs normally require mysql or root OS user accounts. If your cloud service does not give OS accounts, MHA can not be used there.
> MHA does not depend on virtual ip addresses. So even though cloud vendors do not provide many virtual ip addresses, you can use MHA.

# Is there any other solution to do the same thing as MHA? #

> AFAIK, no. If you have one master and only one slave, "some slaves behind the latest slave" situation never happens so automated failover becomes much easier. On cloud environments, Amazon RDS's MultiAZ provides such functionality. But remember that using one slave will cause lots of potential problems.

# Is it possible to promote a specific server to the new master? I do not want to promote some servers because it's not so much stable. #

> Yes. Check [candidate\_master](Parameters#candidate_master.md) and [no\_master](Parameters#no_master.md) parameters.

# Is it possible to do interactive/manual failover with MHA? #
> Yes. Check [masterha\_master\_switch](masterha_master_switch.md) command.

# When does MHA Manager NOT do failover? #

> When one or more slave servers are not alive, or in some situations described later, MHA does not start failover

> "Not Alive" means the followings.

  * Can't connect via MySQL
  * SQL thread can not be started with errors
  * Can't connect via SSH (only when applying differential relay log events is needed)

> For example, if one of the slaves stops with duplicate key errors, MHA does not start failover. This is because such SQL errors can not be solved automatically and manual fixes by DBAs are needed. Master failure caused by MySQL/OS/HW failure does not cause SQL thread stops (unless you use older buggy MySQL versions) so this scenario should not happen.

> If you set [ignore\_fail](Parameters#ignore_fail.md) parameter on each host at the configuration file, MHA starts failover even though the host(s) are not alive.

> MHA also does not start failover in the following cases.

  * When binary log or relay log filtering rules (binlog-do-db, replicate-do-db, etc) are not same between slaves
  * When the last failover failed. In this case, failover error file is created under workdir. Remove it and restarts manual failover
  * When the last failover was done too recently (8 hours by default, can be changed by setting --last\_failover\_minute=(minute)). In this case, MHA Manager does not do failover because it is highly likely that problems can not be solved by just doing failover.


# Which host is NOT selected as a new master? #

> The following hosts are NOT selected as new master.

  * no\_master=1 is set in the configuration file
  * log-bin is not enabled in MySQL
  * Major MySQL version (5.0/5.1/..) is not oldest between all slaves
  * SQL thread delays significantly. If Relay\_Master\_Log\_File on a target slave is behind Master\_Log\_File on the latest slave, or Exec\_Master\_Log\_Pos behinds more than 100,000,000 against Read\_Master\_Log\_Pos on the latest slave, MHA decides that the target slave delays significantly


# Which host is selected as a new master? #

> If hosts meet the above criteria, new master is determined based on the following rules.

  * If you set candidate\_master=1 on some hosts, they will be prioritised.
    * If some of them are the latest (slaves that have received the latest binary log events), the host will be selected as a new master
    * If multiple hosts are the latest, a master host will be determined by "order by section name in the config file". If you have server1, server2, and server3 sections and both server1 and server3 are candidate\_master and the latest, server1 will be selected as a new master.

  * If none of servers set candidate\_master=1, the latest slave server will be the new master. If multiple slaves are the latest, order by section name rule will be applied.

  * If none of the latest slaves can be new master, one of the non-latest slaves will be new master. Order by section name rule will also be applied here.

  * If you set [latest\_priority](Parameters#latest_priority.md) parameter to 0, "whether the slave is the latest or not" does not matter to decide new master at all. If you want to fully control priorities for the new master (you can decide rules in the config file: order by section name), using this parameter may help.

# How to make MHA Manager node itself Highly Available ? #
> Currently monitoring the same master from multiple MHA Manager is NOT supported. So if the MHA Manager goes down, automated master failover can not be started. MHA Manager down does not result in master down, so it's much less serious, but you may want to make MHA Manager highly available as long as possible.
> Current recommended MHA Manager HA solution is using common active/standby solutions like Pacemaker.

# When we add/remove MySQL servers, what should be done? #
> When you add or remove MySQL servers, application config file should be updated in order that new hosts are added or old hosts are deleted. After updating the file, it is recommended restarting MHA Manager.
> [masterha\_conf\_host](masterha_conf_host.md) is useful when you want to this process automated.

# I do not want to write plain text password inside configuration files. #
> Check [init\_conf\_load\_script](Parameters#init_conf_load_script.md) parameter. This parameter is introduced for such purposes.

# How can I set master host in the configuration file? #
> You don't need to set master host in the application configuration file. MHA Manager connects all hosts in the configuration file and automatically identify current master server.

# What if necessary relay logs are purged by purge\_relay\_logs while executing failover? #
> This should never happen. MHA Manager takeks advisory locks on all slaves at the beginning of failover. purge\_relay\_logs client program also takes the same advisory lock on the target slave when executing purge operation.