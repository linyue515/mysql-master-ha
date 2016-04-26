

# Other Solutions and Issues #

# Doing everything manually #
MySQL Replication is asynchronous or semi-synchronous. When master crashes, it is possible that some of slaves have not received the latest relay log events, which means each slave might be in different state each other. Fixing consistency problems manually is not trivial. But without fixing consistency problems, replication might not be able to start (i.e. duplicate key error). It is not uncommon to take more than one hour to restart replication manually.

# Single master and single slave #
If you have one master and only one slave, "some slaves behind the latest slave" situation never happens. When the master crashes, just let applications send all traffics to the new master. So failover is easy.
<pre>
M(RW)<br>
|        --(master crash)-->   M(RW), promoted from S<br>
S(R)<br>
</pre>

But there are lots of very serious issues. First, you can not scale out read traffics. In many cases you may want to run expensive operations on one of slaves such as backups, analytic queries, batch jobs. These might cause performance problems on the slave server. If you have only one slave and the slave is crashed, the master has to handle all such traffics.

Second issue is availability. If the master is crashed, only one server (new master) remains, so it becomes single point of failure. To create a new slave, you need to take an online backup, restore it on the new hardware, and start slave immediately. But these operations normally takes hours (or even more than one day to fully catch up replication) in total. On some critical applications you may not accept that the database becomes single point of failure for such a long time. And taking an online backup on master increases i/o loads significantly so taking backups during the peak time is dangerous.

Third issue is lack of extensibility. For example, when you want to create a read-only database on a remote data center, you need at least two slaves, one is a slave on the local data center, the other is a slave on the remote data center. You can not build this architecture if you have only one slave.

Single slave is actually not enough in many cases.

# Master, one candidate master, and multiple slaves #
Using "one master, one candidate master, and multiple slaves" architecture is also common. Candidate master is a primary target for the new master if the current master crashes. In some cases it is configured as a read-only master: multi-master configuration.
<pre>
M(RW)-----M2(R)                      M(RW), promoted from M2<br>
|                                          |<br>
+----+----+          --(master crash)-->   +-x--+--x-+<br>
S(R)     S2(R)                             S(?)      S(?)<br>
(From which position should S restart replication?)<br>
</pre>


But this does not always work as a master failover solution. When current master crashes, the rest slaves might not have received all relay log events, so fixing consistency issues between slaves are still needed like other solutions.

What if you can not accept consistency problems but want to start service immediately? Just start the candidate master as a new master, and drop all the rest slaves. After that you can create new slaves by taking an online backup from the new master. But this approach has the same problem as the above "Single master and single slave" approach. The rest slaves can not be used for read scaling or redundancy purposes.

This architecture is pretty widely used, but not many people fully understand the potential problems described above. When the current master crashes, slaves become inconsistent or even you can't start replication if you simply let slaves start replication from the new master. If you need guarantee consistency, the rest slaves can't be used. Both approaches have serious disadvantages.


By the way, "using two masters (one is read only) and each master having at least one slaves (like below)" is also possible.
<pre>
M(RW)--M2(R)<br>
|      |<br>
S(R)   S2(R)<br>
</pre>

At least one slave can continue replication when current master crashes, but actually there are not so many users adopting this architecture. The biggest disadvantage is complexity. Three-tier replication(M->M2->S2) is used in this architecture. But managing three-tier replication is not easy. For example, if the middle server (M2:candidate master) crashes, the third-tier slave(S2) can't continue replication. In many cases you have to setup both M2 and S2 again. It is also important that you need at least four servers in this architecture.


# Pacemaker + DRBD #
Using Pacemaker(Heartbeat)+DRBD+MySQL is a very common HA solution. But this solution also has some serious issues.

One issue is cost, especially if you want to run lots of MySQL replication environments. Pacemaker+DRBD is active/standby solution, so you need one passive master server that does not handle any application traffic. The passive server can not be used for read scaling. Typically you need at least four MySQL servers, one active master, one passive master, two slaves (one of the two for reporting etc).

Second issue is downtime. Since Pacemaker+DRBD is active/standby cluster so if the active server crashes, crash recovery happens on the passive server. This might take very long time, especially if you are not using InnoDB Plugin. Even though you use InnoDB Plugin, it is not uncommon to take a few minutes or more to start accepting connections on the standby server. In addition to crash recovery time, warm-up (filling data into buffer pool) takes significant time after the failover, since on the passive server database/filesystem cache is empty. In practice, you need one or more additional slave servers to handle enough read traffics. It is also important that write performance also drops significantly during warm-up time because cache is empty.

Third issue is write performance drops or consistency problems. To make active/passive HA cluster really work, you have to flush transaction logs(binary log and InnoDB log) to disks at every commit. That is, you have to set innodb-flush-log-at-trx-commit=1 and sync-binlog=1. But currently sync-binlog=1 kills write performance because fsync() is serialized in current MySQL(group commit is broken if sync-binlog is 1). In most cases people do not set sync-binlog=1. But if sync-binlog=1 is not set, when the active master crashes, the new master (the previous passive server) might lose some of binary log events that have already been sent to slaves. Suppose the master was crashed and slave A received up to mysqld-bin.000123 position 1500. If binlog data was flushed to disks only up to position 1000, the new master has mysqld-bin.000123 only up to 1000 and creates new binary log mysqld-bin.000124 at start-up. If this happens, slave A can't continue replication because new master doesn’t have mysqld-bin.000123 position 1500.

Fourth issue is complexity. It is actually not trivial to install/configure Pacemaker and DRBD (especially DRBD) for many users. Configuring DRBD often requires re-creating OS partitions in many deployments, which is not easy for many cases. You also need to have enough expertise on DRBD and Linux kernel layer. If you execute a wrong command (i.e. executing drbdadm -- --overwrite-data-of-peer primary on the passive node), it easily breaks live data. It is also important that once disk i/o layer problem happens when using DRBD, fixing the problem is really difficult for most of DBAs.

# MySQL Cluster #
MySQL Cluster is really Highly Available solution, but you have to use NDB storage engine. When you use InnoDB (in most cases), you can't take advantages of MySQL Cluster.

# Semi-Synchronous Replication #
[Semi-Synchronous replication](http://dev.mysql.com/doc/refman/5.5/en/replication-semisync.html) greatly minimizes a risk of "binlog events exist only on the crashed master" situation. This is really helpful to avoid data loss. But Semi-Synchronous replication does not solve all consistency issues. Semi-Synchronous replication guarantees that **at least one** (not all) slaves receive binlog events from the master at commmit. There are still possibilities that some of slaves have not received all binlog events. Without applying differential relay log events from the latest slave to non-latest slaves, slaves can not be consistent each other.

MHA takes care of these consistency issues, so by using both Semi-Synchronous replication and MHA, both "almost no data loss" and "slaves consistency" can be achieved.

# Global Transaction ID #
The purpose of the global transaction id is basically same as what MHA tries to achieve, but it covers more. MHA works with only two tier replication, but global transaction id covers any tier replication environment, so even though second tier slave fails, you can recover third tier slave. Check [Google's global transaction id project](http://code.google.com/p/google-mysql-tools/wiki/GlobalTransactionIds) for details.

Starting from MySQL 5.6, GTID was supported. Oracle's official tool [mysqlfailover](http://dev.mysql.com/doc/mysql-utilities/1.3/en/mysqlfailover.html) supports master failover with GTID. Starting from MHA version 0.56, MHA also supports failover based on GTID. MHA automatically detects whether mysqld is running with GTID or not, and if GTID is enabled, MHA does failover with GTID. If not, MHA does traditional failover with relay logs.