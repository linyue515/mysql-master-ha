# masterha\_manager: Command to run MHA Manager #

> MHA Manager can be started by executing masterha\_manager command.
```
  # masterha_manager --conf=/etc/conf/masterha/app1.cnf
```

> masterha\_manager takes below arguments.

## Common arguments ##

  * --conf=(config file path)
> Application and local scope configuration file. This argument is mandatory.

  * --global\_conf=(global config file path)
> Global scope configuration file. Default is /etc/masterha\_default.cnf

  * --manager\_workdir, --workdir
> Same as [manager\_workdir](Parameters#manager_workdir.md) parameter.

  * --manager\_log, --log\_output
> Same as [manager\_log](Parameters#manager_log.md) parameter.

## Monitoring specific arguments ##

  * --wait\_on\_monitor\_error=(seconds)
> If an error happens during monitoring, masterha\_manager sleeps wait\_on\_monitor\_error seconds and exits. The default is 0 seconds(not waiting). This functionality is mainly introduced for doing automated master monitoring and failover from daemon scripts. It is pretty reasonable for waiting for some time on errors before restarting monitoring again.

  * --ignore\_fail\_on\_start
> By default, master monitoring (not failover) process stops if one or more slaves are down, regardless of "ignore\_fail" parameter setting. By setting --ignore\_fail\_on\_start, master monitoring does not stop if ignore\_fail marked slaves are down.

## Failover specific arguments ##

  * --last\_failover\_minute=(minutes)
> If the previous failover was done too recently (8 hours by default), MHA Manager does not do failover because it is highly likely that problems can not be solved by just doing failover. The purpose of this parameter is to avoid ping-pong failover problems. You can change time criteria by changing this parameter. The default is 480 (8 hours).
> If --ignore\_last\_failover is set, this step is ignored.

  * --ignore\_last\_failover
> If the previous failover failed, MHA does not start failover because the problem might happen again. The normal step to start failover is manually remove failover error file which is created under (manager\_workdir)/(app\_name).failover.error .
> By setting --ignore\_last\_failover, MHA continues failover regardless of the last failover status.

  * --wait\_on\_failover\_error=(seconds)
> If an error happens during failover, MHA Manager sleeps wait\_on\_failover\_error seconds and exits. The default is 0 seconds (not waiting). This functionality is mainly introduced for doing automated master monitoring and failover from daemon scripts. It is pretty reasonable for waiting for some time on errors before restarting monitoring again.

  * --remove\_dead\_master\_conf
> When this option is set, if failover finishes successfully, MHA Manager automatically removes the section of the dead master from the configuration file. For example, if the dead master's hostname is host1 and it belongs to the section of server1, the entire part of the server1 will be removed from the configuration file.
> By default, the configuration file is not modified at all. After MHA finishes failover, the section of the dead master still exists. If you start masterha\_manager immediately (this includes automatic restart from any daemon program), masterha\_manager stops with an error that "there is a dead slave" (previous dead master). You might want to change this behavior especially if you want to continuously monitor and failover MySQL master automatically. In such cases, --remove\_dead\_master\_conf argument is helpful.