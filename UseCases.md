
# Typical Use cases #

## Where to deploy Manager ##

  * Dedicated Manager server and multiple MySQL (master,slaves) servers

> Since MHA Manager uses very little CPU/Memory resources, you can manage lots of (master, slaves) pairs from single MHA Manager. It is even possible to manage 100+ pairs from single manager server.

  * Running MHA Manager on one of MySQL slaves

> If you have only one (master, slaves) pair, you may not like allocating dedicated hardware for MHA Manager because it adds relatively high costs. In such cases, running MHA Manager on one of slaves makes sense. Note that current version of MHA Manager connects to MySQL slave server via SSH even though the MySQL server is located on the same host as MHA Manager, so you need to enable SSH public key authentication from the same host.

## Replication configuration ##

### Single master, multiple slaves ###

<pre>
M(RW)                            M(RW), promoted from S1<br>
|                                             |<br>
+------+------+        --(master crash)-->     +-----+-----+<br>
S1(R) S2(R)   S3(R)                           S2(R)        S3(R)<br>
</pre>

> This is the most common replication settings. MHA works very well here.


### Single master, multiple slaves (one on remote datacenter) ###

<pre>
M(RW)                                M(RW), promoted from S1<br>
|                                                 |<br>
+------+---------+         --(master crash)-->     +-----+------+<br>
S1(R) S2(R)     Sr(R,no_master=1)                   S2(R)         Sr(R,no_master=1)<br>
</pre>

> In many cases you want to deploy at least one slave server on a remote datacenter. When the master crashes, you may not want to promote the remote slave to the new master, but let one of other slaves running on the local datacenter become the new master. MHA supports such requirements. Setting [no\_master=1](Parameters#no_master.md) in the [configuration file](Configuration.md) makes the slave never becomes new master.

### Single master, multiple slaves, one candidate master ###

<pre>
M(RW)-----S0(R,candidate_master=1)   M(RW), promoted from S0<br>
|                                          |<br>
+----+----+          --(master crash)-->   +----+----+<br>
S1(R)     S2(R)                             S1(R)      S2(R)<br>
</pre>

> In some cases you may want to promote a specific server to the new master if the current master crashes. In such cases, setting [candidate\_master=1](Parameters#candidate_master.md) in the [configuration file](Configuration.md) will help.


### Multiple masters, multiple slaves ###

<pre>
M(RW)<--->M2(R,candidate_master=1)   M(RW), promoted from M2<br>
|                                          |<br>
+----+----+          --(master crash)-->   +----+----+<br>
S(R)     S2(R)                             S1(R)      S2(R)<br>
</pre>

> In some cases you may want to use multi-master configurations, and you may want to make the read-only master the new master if the current master crashes.
> MHA Manager supports multi-master configurations as long as all non-primary masters (M2 in this figure) are read-only.


### Three tier replication ###

<pre>
M(RW)                                   M(RW), promoted from S1<br>
|                                                 |<br>
+------+---------+         --(master crash)-->     +-----+------+<br>
S1(R) S2(R)     Sr(R)                              S2(R)       Sr(R)<br>
|                                              |<br>
+                                              +<br>
Sr2                                            Sr2<br>
</pre>

> In some cases you may want to use three-tier replication like this. MHA can still be used for master failover. In the [configuration file](Configuration.md), manage the master and all second-tier slaves (in this figure, add M,S1,S2 and Sr in the MHA config file, but do not add Sr2). If the current master (M) fails, MHA automatically promotes one of the second-tier slaves(S1,S2,Sr, and you can also set priorities) to the new master, and recover the rest second-tier slaves. The third tier slave(Sr2) is not managed by MHA, but as long as Sr (Sr2's master) is alive, Sr2 can continue replication without changing anything.

> If Sr crashes, Sr2 can't continue replication because Sr2's master is Sr. MHA can NOT be used to recover Sr2. This is where [support for global transaction id](http://code.google.com/p/google-mysql-tools/wiki/GlobalTransactionIds) is desired. Hopefully this is less serious than master crash.


### Three tier replication, multi-master ###

<pre>
M1(host1,RW) <-----------------> M2(host2,read-only)<br>
|                                |<br>
+-----+--------+                       +<br>
S1(host3,R)    S2(host4,R)             S3(host5,R)<br>
<br>
<br>
=> After failover<br>
<br>
M2(host2,RW)<br>
|<br>
+--------------+--------------------------+<br>
S1(host3,R)    S2(host4,R)             S3(host5,R)<br>
<br>
</pre>

> This structure is also supported. In this case, host5 is a third-tier slave, so MHA does not manage host5(MHA does not execute CHANGE MASTER on host5 when the primary master host1 fails). When current master host1 is down, host2 will be new master, so host5 can keep replication from host2 without doing anything.
> Here are [configuration](Configuration.md) file example.

```
[server default]
multi_tier_slave=1

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

[server5]
hostname=host5
```


## Managing Master IP address ##

> On common HA environments, many cases people allocate one virtual IP address on a master. If the master crashes, HA software like Pacemaker takes over the virtual IP address to a standby server.

> Another common approach is creating a global catalog database that has all mappings between application names and writer/reader IP addresses (i.e. {app1\_writer, 192.168.0.1}, {app2\_writer, 192.168.0.2}, …), instead using virtual IP addresses. In this case, you need to update the catalog database when the current master dies.

> Both approaches have advantages and disadvantages. MHA does not force one approach, but let users can use any IP address failover solution. MHA has an option to call an external script to inactivate/activate writer IP address, by setting [master\_ip\_failover\_script](Parameters#master_ip_failover_script.md) parameter at manager's configuration file. You can update a catalog database, takeover virtual IP address, or do whatever you want in the script. You can also use an existing HA software like Pacemaker to do IP failover. In this case MHA itself does not do IP failover.


## Using together with Semi-Synchronous Replication ##

> Though MHA tries to save binary logs from the crashed master, it is not always possible. For example, if the crashed master dies with H/W failure or can not be reachable via SSH, MHA can not save binary logs and has to do failover without applying binary log events that exist on the crashed master only. This will result in losing the latest data.

> Using Semi-Synchronous replication greatly reduces the risk of such data loss. MHA works with Semi-Synchronous replication, since it is based on MySQL replication mechanism. It is worth to note that if only one of the slaves have received the latest binary log events, MHA can apply the events to all other slaves so they can be consistent each other.