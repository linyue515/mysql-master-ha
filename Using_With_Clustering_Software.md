
# Using with clustering software #

If you use a virtual IP address for a master, you might already use clustering software like Pacemaker. If you are familiar with existing clustering tools, you may want to use them to manage virtual ip address, rather than doing everything in MHA. MHA can be used just for (non-interactive) failover, so you can use clustering software and MHA together for master failover.

Clustering software manages a virtual IP address on the master and does node fencing. MHA focuses on promoting a new master and fixing slave consistency.

Here is a brief example for configuring Pacemaker (Heartbeat v1 mode).

```
  # /etc/ha.d/haresources on host2
  host2  failover_start  IPaddr::192.168.0.3
```


# failover\_start script example

```
  
  start)
  `masterha_master_switch --master_state=dead --interactive=0 --wait_on_failover_error=0 --dead_master_host=host1 --new_master_host=host2`
  exit
  
  stop)
  # do nothing
```


# Application configuration file:

```
  [server1]
  hostname=host1
  candidate_master=1
  
  [server2]
  hostname=host2
  candidate_master=1
  
  [server3]
  hostname=host3
  no_master=1
```


Since data files are not shared, data recources are not needed to be managed by clustering software or DRBD. For this purpose, clustering software just invokes masterha\_master\_switch script and takes over virtual IP address. You can do the same thing by handmade scripts etc.