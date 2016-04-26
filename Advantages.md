
# Advantages of MHA #
## Master failover and slave promotion can be done very quickly ##
> MHA normally can do failover in seconds (9-12 seconds to detect master failure, optionally 7-10 seconds to power off the master machine to avoid split brain, a few seconds for applying differential relay logs to the new master, so total downtime is normally 10-30 seconds), as long as slaves does not delay replication seriously. After revoring the new master, MHA recovers the rest slaves in parallel. Even though you have tens of slaves, it does not affect master recovery time, and you can recover slaves very quickly.

> DeNA uses MHA on 150+ {master, slaves} environments. When one of the master  crashed, MHA completed failover in 4 seconds. Doing failover in 4 seconds is never possible with traditional active/passive clustering solution.

## Master crash does not result in data inconsistency ##

> When the current master crashes, MHA automatically identifies differential relay log events between slaves, and applies to each slave. So finally all slaves can be in sync, as long as all slave servers are alive.
> By using together with Semi-Synchronous Replication, (almost) no data loss can also be guaranteed.

## No need to modify current MySQL settings (MHA works with regular MySQL (5.0 or later)) ##

> One of the most important design principles of MHA is to make MHA easy to use as long as possible. MHA works with existing traditional MySQL 5.0+ master-slaves replication environments. Though many other HA solutions require to change MySQL deployment settings, MHA does not force such tasks for DBAs. MHA works with the most common two-tier single master and multiple slaves environments. MHA works with both asynchronous and semi-synchronous MySQL replication. Starting/Stopping/Upgrading/Downgrading/Installing/Uninstalling MHA can be done without changing (including starting/stopping) MySQL replication. When you need to upgrade MHA to newer versions, you don't need to stop MySQL. Just replace with newer MHA versions and restart MHA Manager is fine.

> MHA works with normal MySQL versions starting from MySQL 5.0. Some HA solutions require special MySQL versions (i.e. MySQL Cluster, MySQL with Global Transaction ID, etc), but you may not like to migrate applications just for master HA. In many cases people have already deployed many legacy MySQL applications and they don't want to spend too much time to migrate to different storage engines or newer bleeding edge distributions just for master HA. MHA works with normal MySQL versions including 5.0/5.1/5.5 so you don't need to migrate.

## No need to increase lots of servers ##

> MHA consists of MHA Manager and MHA Node. MHA Node runs on the MySQL server when failover/recovery happens so it doesn't require additional server. MHA Manager normally runs on a dedicated server so you need to add one (or two for HA) server(s), but MHA Manager can monitor lots of (even 100+) masters from single server, so the total number of servers is not increased so much. Note that it is even possible to run MHA Manager on one of slave servers. In this case total number of servers is not increased at all.

## No performance penalty ##

> MHA works with regular asynchronous or semi-synchronous MySQL replication. When monitoring master server, MHA just sends ping packets to master every N seconds (default 3) and it does not send heavy queries. You can expect as fast performance as regular MySQL replication.

## Works with any storage engine ##

> MHA works with any storage engines as long as MySQL replication works, not limited to InnoDB (crash-safe, transactional storage engine). Even though you use legacy MyISAM environments that are not easy to migrate, you can use MHA.