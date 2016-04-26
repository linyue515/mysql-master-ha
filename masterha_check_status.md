## Checking Manager Status with masterha\_check\_status ##

> MHA Manager provides a command line client "masterha\_check\_status" to check whether the Manager monitors MySQL master correctly or not. Here is an example.

```
  $ masterha_check_status --conf=/path/to/app1.cnf
  app1 (pid:8368) is running(0:PING_OK), master:host1
  $ echo $?
  0
```

> When MHA Manager is successfully monitoring the MySQL master, status code (exit code) 0 should be returned like above.

> All status codes and descriptions are listed below.


|**Status Code(Exit code)**|**Status String**|**Description**|
|:-------------------------|:----------------|:--------------|
|0                         |PING\_OK         |Master is running and MHA Manager is monitoring. Master state is alive.|
|1                         |---              |Unexpected error happened. For example, config file does not exist. If this error happens, check arguments are valid or not.|
|2                         |NOT\_RUNNING     |MHA Manager is not running. Master state is unknown.|
|3                         |PARTIALLY\_RUNNING|MHA Manager main process is not running, but child processes are running. This should not happen and should be investigated. Master state is unknown.|
|10                        |INITIALIZING\_MONITOR|MHA Manager is just after startup and initializing. Wait for a while and see how the status changes. Master state is unknown.|
|20                        |PING\_FAILING    |MHA Manager detects ping to master is failing. Master state is maybe down.|
|21                        |PING\_FAILED     |MHA Manager detects either a) ping to master failed three times, b) preparing for starting master failover. Master state is maybe down.|
|30                        |RETRYING\_MONITOR|MHA Manager internal health check program detected that master was not reachable from manager, but after double check MHA Manager verified the master is alive, and currently waiting for retry. Master state is very likely alive.|
|31                        |CONFIG\_ERROR    |There are some configuration problems and MHA Manager can't monitor the target master. Check a logfile for detail. Master state is unknown.|
|32                        |TIMESTAMP\_OLD   |MHA Manager detects that ping to master is ok but status file is not updated for a long time. Check whether MHA Manager itself hangs or not. Master state is unknown.|
|50                        |FAILOVER\_RUNNING|MHA Manager confirms that master is down and running failover. Master state is dead.|
|51                        |FAILOVER\_ERROR  |MHA Manager confirms that master is down and running failover, but failed during failover. Master state is dead.|