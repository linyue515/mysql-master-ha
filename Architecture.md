
# Architecture of MHA #
> When a master crashes, MHA recovers rest slaves as below.

![http://4.bp.blogspot.com/-AIInutsrkAk/TiF5kAcIzQI/AAAAAAAAAFg/jW1SnzM8T_E/s600/mha_recovery_procedure.png](http://4.bp.blogspot.com/-AIInutsrkAk/TiF5kAcIzQI/AAAAAAAAAFg/jW1SnzM8T_E/s600/mha_recovery_procedure.png)

Fig: Steps for recovery

> Basic algorithms are described in the [slides](http://www.slideshare.net/matsunobu/automated-master-failover) presented at the MySQL Conference and Expo 2011, especially from page [no.13](http://www.slideshare.net/matsunobu/automated-master-failover/13) to [no.34](http://www.slideshare.net/matsunobu/automated-master-failover/34).

> In relay log files on slaves, master's binary log positions are written at "end\_log\_pos" sections ([example](http://www.slideshare.net/matsunobu/automated-master-failover/18)). By comparing the latest end\_log\_pos between slaves, we can identify which relay log events are not sent to each slave. MHA internally recovers slaves (fixes consistency issues) by using this mechanism. In addition to basic algorithms covered in the [slides](http://www.slideshare.net/matsunobu/automated-master-failover)  at the MySQL Conf 2011, MHA does some optimizations and enhancements, such as generating differential relay logs very quickly (indenpendent from relay log file size), making recovery work with row based formats, etc.

## MHA Components ##
> MHA consists of MHA Manager and MHA Node as below.

![http://1.bp.blogspot.com/-wCKF5uopGNI/TiF4dA2df6I/AAAAAAAAAFY/AHScjL84JF4/s600/components_of_mha.png](http://1.bp.blogspot.com/-wCKF5uopGNI/TiF4dA2df6I/AAAAAAAAAFY/AHScjL84JF4/s600/components_of_mha.png)

Fig: MHA components

> MHA Manager has manager programs such as monitoring MySQL master, controlling master failover, etc.

> MHA Node has failover helper scripts such as parsing MySQL binary/relay logs, identifying relay log position from which relay logs should be applied to other slaves, applying events to the target slave, etc. MHA Node runs on each MySQL server.

> When MHA Manager does failover, MHA manager connects MHA Node via SSH and executes MHA Node commands when needed.

## Custom Extensions ##

> MHA has a couple of extention points. For example, MHA can call any custom script to update master's ip address (updating global catalog database that manages master's ip address, updating virtual ip, etc). This is because how to manage IP address depends on users' environments and MHA does not want to force one approach.

> Current extension points are as below. MHA Manager package includes sample scripts.
  * [secondary\_check\_script](Parameters#secondary_check_script.md): For checking master availability from multiple network routes
  * [master\_ip\_failover\_script](Parameters#master_ip_failover_script.md): For updating master's ip address used from applications
  * [shutdown\_script](Parameters#shutdown_script.md): For forcing shutdown the master
  * [report\_script](Parameters#report_script.md): For sending reports
  * [init\_conf\_load\_script](Parameters#init_conf_load_script.md): For loading initial configuration parameters
  * [master\_ip\_online\_change\_script](Parameters#master_ip_online_change_script.md): For updating master ip address. This is not used for master failover, but used for [online master switch](http://code.google.com/p/mysql-master-ha/wiki/masterha_master_switch#Scheduled(Online)_Master_Switch)