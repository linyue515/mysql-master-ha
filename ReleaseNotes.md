

# MHA Manager #
### Changes in Manager 0.57 (May 31 2015) ###
> Bug Fix:
    * Code cleanup: Disconnecting properly on connection checks
    * Masking plaintext passwords
    * Ignore event scheduler when checking long updates
    * Use InnoDB for ping insert

### Changes in Manager 0.56 (Apr 1 2014) ###
> New Features:
    * Supporting MySQL 5.6 GTID (If GTID and auto position is enabled, MHA automatically does failover using GTID syntax)
    * Supporting MySQL 5.6 Multi-Threaded slave
    * Supporting MySQL 5.6 binlog checksum
    * Supporting mysqlbinlog streaming hosts. In some cases, people use binlog archiving servers via streaming mysqlbinlog. MHA supports that by `[binlog]` section. This works with GTID. If `[binlog]` is added, MHA checks the hosts and if it has more binlog events than other slaves, MHA uses these events for recovery.
    * Supporting custom mysql and mysqlbinlog location
    * Adding ping\_type=INSERT for checking connectivity for the master. This is useful if master does not accept any writes (i.e. disk error)
    * Added --orig\_master\_is\_new\_slave, --orig\_master\_ssh\_user and --new\_master\_ssh\_user for master\_ip\_online\_change\_script

### Changes in Manager 0.55 (Dec 12 2012) ###
> Bug Fix:
    * MHA does not start if MySQL password is empty. This was a regression bug in 0.54 ([details](http://code.google.com/p/mysql-master-ha/issues/detail?id=44))

### Changes in Manager 0.54 (Dec 1 2012) ###
> Dependency:
    * MHA Manager 0.54 requires MHA Node 0.54. And MHA Node 0.54 can not be invoked from MHA Manager version 0.53 or older.

> New features:
    * Supporting --new\_master\_user and --new\_master\_port for master\_ip\_failover\_script, and supporting --orig\_master\_user, --orig\_master\_password, --new\_master\_user and --new\_master\_password for master\_ip\_online\_change\_script
    * Adding "ssh\_connection\_timeout" parameter. Default is 5 seconds. Before adding this parameter timeout was hard coded.
    * Adding --ignore\_fail\_on\_start command line argument so that masterha\_monitor does not stop if ignore\_fail marked slaves are down
    * Supporting --skip\_change\_master command line argument. By setting this argument on master failover, MHA exits after applying differential relay logs, and skipping CHANGE MASTER and START SLAVE
    * Adding --skip\_disable\_read\_only command line argument. By passing this parameter, MHA skips executing SET GLOBAL read\_only=0 on the new master. This may be useful when you want to manually disable this.

> Bug fixes:
    * MHA stops with errors (UUV) when using Log::Dispatch version 2.30 or higher
    * MySQL user and password should be escaped properly (they were not properly handled if special characters were included)
    * masterha\_check\_ssh should not try to read from files if not exist


### Changes in Manager 0.53 (Jan 9 2012) ###
> New features:
    * [Manager-Issue#12](https://github.com/yoshinorim/mha4mysql-manager/issues/12) Supporting RESET SLAVE ALL from MySQL 5.5.16
    * [Manager-Issue#15](https://github.com/yoshinorim/mha4mysql-manager/issues/15) Supporting "skip\_reset\_slave" parameter to avoid running CHANGE MASTER TO on the promoted slave
    * [Manager-Issue#18](https://github.com/yoshinorim/mha4mysql-manager/issues/18) Checking master's ping status via CONNECT, in addition to SELECT
    * Supporting --check\_only for online master switch / [patch](https://github.com/yoshinorim/mha4mysql-manager/commit/b7ce7c7444220f2f4209dddc2c959ada027d7466)
    * Supporting "mysql --binary-mode" from MySQL 5.6.3 / [patch](https://github.com/yoshinorim/mha4mysql-manager/commit/09cf63f7ad0930bf6a9955205a3cd6ee51e76308)
    * Supporting ssh\_host and ssh\_port parameters / [patch](https://github.com/yoshinorim/mha4mysql-manager/commit/0e0bb1bccfced545dfa45e2747d8d55bf0a7b444)
    * Supporting ssh\_options parameter / [patch](https://github.com/yoshinorim/mha4mysql-manager/commit/fe6b425bf323e577a065b963151aa7d4c14bd856)
    * When doing online master switch, MHA checks whether long queries are running on the new master. This is important to reduce workloads on the new master. Query time limit can be controlled via --running\_seconds\_limit. / [patch](https://github.com/yoshinorim/mha4mysql-manager/commit/4bb9953fce0d11e1c22be406b24fb9904a4b0b3e)
    * When executing SIGINT on online master switch, MHA tries to disconnect established connections via MHA. / [patch](https://github.com/yoshinorim/mha4mysql-manager/commit/2996e082846c4c6db5a3f2182a211e0b07554baa)
    * Additionally checking replication filtering rules on online master switch / [patch](https://github.com/yoshinorim/mha4mysql-manager/commit/051f362bf3442ac5892f1f65afc307382d13b5e2)

> Bug fixes:
    * [Manager-Issue#13](https://github.com/yoshinorim/mha4mysql-manager/issues/13) MHA Manager looks for relay-log.info in wrong location
    * [Manager-Issue#14](https://github.com/yoshinorim/mha4mysql-manager/issues/14) Wrong option for master\_ip\_failover\_script
    * [Manager-Issue#17](https://github.com/yoshinorim/mha4mysql-manager/issues/17) Timeout settings for SSH connection health check does not always work
    * Modifying a rpm spec file to create valid rpm package for 64bit RHEL6 / [patch](https://github.com/yoshinorim/mha4mysql-manager/commit/9d149aa5ea3fdea72615d12f6b83bb7e795a5e5a)
    * [Manager-Issue#19](https://github.com/yoshinorim/mha4mysql-manager/issues/19) Forcing more strict ssh checking. Originally MHA checks master's reachability by just connecting via SSH and exiting with return code 0. This in some cases does not work especially if SSH works but data files are not accessible. In this fix, MHA checks master's ssh reachability by executing save\_binary\_logs command (dry run). MHA Client also needs to be updated to 0.53.
    * Zombie process remains on master ping timeout / [patch](https://github.com/yoshinorim/mha4mysql-manager/commit/3de2e93078b63751f19464e63b05823160518c84)
    * Do not execute SET GLOBAL read\_only=(0|1) if not needed / [patch](https://github.com/yoshinorim/mha4mysql-manager/commit/06dd340ecba89494bda82c3b548723e0cad14b10)
```
     SET GLOBAL read_only=(0|1) conflicts with long running queries
    and will block all updates (including from root and SQL Thread)
    until conflicting queries finish. In theory this should happen
    only when setting from 0 to 1, and 5.5 works correcty,
    but current 5.1 locks anytime (including 0->0, 1->1, 1->0).
    This is dangerous and do not run SET GLOBAL read_only=(0|1)
    if not needed.
    MHA should execute SET GLOBAL read_only in the below cases:
    - Online master switch: SET GLOBAL read_only=1 on the current master
    (from master_ip_online_change script)
    - Failover and online master switch: SET GLOBAL read_only=0
    on the new master if current read_only value is 1

    On online master switch, FLUSH NO_WRITE_TO_BINLOG TABLES
    will be executed (manually) beforehand, so this issue will
    rarely happen.
```

### Changes in Manager 0.52 (Sep 16 2011) ###
> Package name change:
    * Package name has changed from "MySQL-MasterHA-Manager-X.Y" to "mha4mysql-manager-X.Y". This is to avoid potential trademark issues.

> New features:
    * [Manager-Issue#5](https://github.com/yoshinorim/mha4mysql-manager/issues/5) Supporting multi-master configuration

> Bug fixes:
    * [Manager-Issue#6](https://github.com/yoshinorim/mha4mysql-manager/issues/6) masterha\_check\_repl may die with UUV when a master dies
    * [Manager-Issue#8](https://github.com/yoshinorim/mha4mysql-manager/issues/8) MHA Manager does not start if Log::Dispatch version is 2.22 or lower
    * [Manager-Issue#10](https://github.com/yoshinorim/mha4mysql-manager/issues/10) @@global.relay\_log\_purge should be preserved on change master
    * [Manager-Issue#11](https://github.com/yoshinorim/mha4mysql-manager/issues/11) Manager aborts by UUV if errstr is empty (rare cases)

> Misc:
    * [Manager-Issue#7](https://github.com/yoshinorim/mha4mysql-manager/issues/7) Connection checks are done in parallel. Failover can be done much faster when multiple MySQL servers are not reachable.
    * Followed Perl Best Practices

### Changes in Manager 0.51 (Aug 18 2011) ###
> Bug fixes:
    * [Manager-Issue#4](http://github.com/yoshinorim/mha4mysql-manager/issues/4) Some tentative working files were written in a current directory, not working directory
    * [Manager-Issue#3](http://github.com/yoshinorim/mha4mysql-manager/issues/3) Checking slave state string is incorrect for MySQL 5.5+. This might stop slave recovery.
    * [Manager-Issue#2](http://github.com/yoshinorim/mha4mysql-manager/issues/2) Wrong exit code on masterha\_manager
    * [Manager-Issue#1](http://github.com/yoshinorim/mha4mysql-manager/issues/1) When multiple servers with the same hostname and different ports are defined, migration information is not printed correctly

> Misc:
    * Public test cases are added.


### Changes in Manager 0.50 (Jul 23 2011) ###
> Initial Public Release

# MHA Node #
### Changes in Node 0.57 (May 31 2015) ###
  * Create/delete test table with MyISAM engine instead of InnoDB

### Changes in Node 0.56 (Apr 1 2014) ###
  * 0.55 was skipped
  * Supporting MySQL 5.6 binlog checksum
  * Added --socket and --defaults-file for purge\_relay\_logs

### Changes in Node 0.54 (Dec 1 2012) ###
> Dependency:
    * MHA Manager 0.54 requires MHA Node 0.54. And MHA Node 0.54 can not be invoked from MHA Manager version 0.53 or older.

> Bug fixes:
    * MySQL user and password should be escaped properly (they were not properly handled if special characters were included)

### Changes in Node 0.53 (Jan 9 2012) ###
> New features:
    * [Issue#3](https://github.com/yoshinorim/mha4mysql-node/issues/3) Supporting "mysql --binary-mode" from MySQL 5.6.3
    * Supporting ssh\_host and ssh\_port parameters
    * Supporting ssh\_options parameter

> Bug fixes:
    * [Issue#2](https://github.com/yoshinorim/mha4mysql-node/issues/2) MHA Manager looks for relay-log.info in wrong location
    * [Issue#4](https://github.com/yoshinorim/mha4mysql-node/issues/4) Forcing more strict ssh checking.


### Changes in Node 0.52 (Sep 16 2011) ###
> Package name change:
    * Package name has changed from "MySQL-MasterHA-Node-X.Y" to "mha4mysql-node-X.Y". This is to avoid potential trademark issues.

> Misc:
    * Followed Perl Best Practices
  * Version 0.51 has skipped.


### Changes in Node 0.50 (Jul 23 2011) ###
> Initial Public Release

> RPM packages:
    * [Node-Issue#1](http://github.com/yoshinorim/mha4mysql-node/issues/1) Node rpm package should not depend on MODULE\_COMPAT
    * RPM packages for RH4,5,6