

# Overview #

> MHA performs automating master failover and slave promotion with minimal downtime, usually within 10-30 seconds. MHA prevents replication consistency problems and saves on expenses of having to acquire additional servers. All this with zero performance degradation, no complexity (easy-to-install) and requiring no change to existing deployments.

> MHA also provides scheduled online master switching, safely changing the currently running master to a new master, within mere seconds (0.5-2 seconds) of downtime (blocking writes only).

> MHA provides the following functionality, and can be useful in many deployments in which high availability, data integrity and near non-stop master maintenance are required.
    * Automated master monitoring and failover
> MHA can monitor MySQL masters in an existing replication environment, performing automatic master failover upon detection of master failure. MHA guarantees the consistency of all the slaves by identifying differential relay log events from the most current slave and applying them to all the other slaves, including those slaves which still haven't received the latest relay log events. MHA can normally perform failover in a matter of seconds: 9-12 seconds to detect master failure, optionally 7-10 seconds to power off the master machine to avoid split brain, and a few seconds to apply differential relay logs to the new master. Total downtime is normally 10-30 seconds. A specific slave can be designated as a candidate master (setting priorities) in a configuration file. Since MHA maintains consistency between slaves, any slave can be promoted to become the new master. Consistency problems, which would ordinarily cause sudden replication failure, will not occur.
    * Interactive (manually initiated) Master Failover
> MHA can be configured for manually initiated (non-automatic), interactive failover, without monitoring the master.
    * Non-interactive master failover
> Non-interactive, automatic master failover without monitoring the master is also supported. This feature is especially useful when MySQL master software monitoring is already in use. For example, you can use [Pacemaker(Heartbeat)](http://www.linux-ha.org/wiki/Pacemaker) for detecting master failure and virtual IP address takeover, while using MHA for master failover and slave promotion.
    * Online switching master to a different host
> It is often necessary to migrate an existing master to a different machine, like when the current master has H/W RAID controller or RAM problems, or when you want to replace it with a faster machine, etc. This is not a master crash, but scheduled master maintenance is required. Scheduled master maintenance should be done as quickly as possible, since it entails partial downtime (master writes are disabled). On the other hand, you should block/kill current running sessions very carefully because consistency problems between different masters may occur (i.e "updating master1, updating master 2, committing master1, getting error on committing master 2" will result in data inconsistency). Both fast master switch and graceful blocking writes are required.

> MHA provides graceful master switching within 0.5-2 seconds of writer blockage. 0.5-2 seconds of writer downtime is often acceptable, so you can switch masters even without allocating a scheduled maintenance window. Actions such as upgrading to higher versions, faster machine, etc. become much easier.


# Difficulties of Master Failover #
> Master Failover is not as trivial as it might seem. Take the most typical MYSQL deployment case of a single master with multiple slaves. If the master crashes, you need to pick one of the latest slaves, promote it to the new master, and let other slaves start replication from the new master. This is actually not trivial. Even when the most current slave can be identified, it is likely that the other slaves have not yet received all their binary log events. Those slaves will lose transactions if connected to the new master upon commencement of replication. This will cause consistency problems. To avoid those consistency problems, the lost binlog events (which haven't yet reached all the slaves) need to be identified and applied to each slave in turn prior to initiating replication on the new (promoted) master. This operation can be very complex and difficult to perform manually. This is illustrated in my presentation at the MySQL Conference and Expo 2011 [slides](http://www.slideshare.net/matsunobu/automated-master-failover) (especially in [p.10](http://www.slideshare.net/matsunobu/automated-master-failover/10) as below).

![http://1.bp.blogspot.com/-vz3yQMLk5Eo/TilhRCi5ZqI/AAAAAAAAAFo/y08RHXsituc/s600/mha-problem.png](http://1.bp.blogspot.com/-vz3yQMLk5Eo/TilhRCi5ZqI/AAAAAAAAAFo/y08RHXsituc/s600/mha-problem.png)

> Fig: Master Failover: What makes it difficult?

> Currently most MySQL Replication users have no choice but to perform failover manually on master crashes. One or more hours of downtime are not uncommon to complete failover. It is likely that not all slaves will have received the same relay log events, resulting in consistency problems later which will have to be corrected later. Even though master crashes are infrequent, they can be very painful when they occur.

> MHA aims to fully automate master failover and recovery procedures as quickly as possible, without any passive (standby) machine. Recovery includes determining the new master, identifying differential relay log events between slaves, applying necessary events to the new master, syncing the other slaves and have them start replication from the new master. MHA normally can to failover in 10-30 seconds of downtime (10 seconds to detect master failure, optionally 7-10 seconds to power off the master machine to avoid split brain, a few seconds or more for recovery), depending on replication delay.

> MHA provides both automated and manual failover commands. The automated failover command "[masterha\_manager](masterha_manager.md) (MHA Manager)" consists of master monitoring and master failover. [masterha\_manager](masterha_manager.md) permanently monitors the master server's availability. If MHA Manager cannot reach the master server, it automatically starts non-interactive failover procedures.

> The manual failover command "[masterha\_master\_switch](masterha_master_switch.md)" initially checks to see that master is in fact dead. If the master is really dead, [masterha\_master\_switch](masterha_master_switch.md) picks one of the slaves as a new master (you can choose a preferred master), and initiates recovery and failover. Internally it does much more, but you execute only one command, without having to perform complex master recovery and failover operations on your own.

# Existing solutions and issues #
> See [Other\_HA\_Solutions](Other_HA_Solutions.md) page for details.

# Architecture of MHA #
> See [Architecture](Architecture.md) page for details.

# Advantages of MHA #
> See [Advantages](Advantages.md) page for details.

# Use Cases #
> See [UseCases](UseCases.md) page for details.