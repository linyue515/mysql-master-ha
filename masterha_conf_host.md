# masterha\_manager: A helper script to add/remove host entries from configuration file #

> In some cases, you might want to add/remove host entries from configuration file automatically. For example, when you setup new slave server, instead of manually editing the config file, you can simply add new host entry like below.

```
  # masterha_conf_host --command=add --conf=/etc/conf/masterha/app1.cnf --hostname=db101
```

> Then the following lines will be added to the conf file.

```
 [server_db101]
 hostname=db101
```

> Note that all comments, white spaces etc are trimmed in the new configuration file.


> You can add several parameters in the config file by passing --param parameters, separated by semi-colon(;).
```
  # masterha_conf_host --command=add --conf=/etc/conf/masterha/app1.cnf --hostname=db101 --block=server100 --params="no_master=1;ignore_fail=1"
```

> The following lines will be added to the conf file.
```
 [server100]
 hostname=db101
 no_master=1
 ignore_fail=1
```

> You can also remove specified block. The below command will remove the etire block [server100](server100.md).

```
  # masterha_conf_host --command=delete --conf=/etc/conf/masterha/app1.cnf --block=server100
```


> masterha\_conf\_host takes below arguments.

# Common arguments

  * --command=(add|delete)
> To add(--command=add) or delete(--command=delete) a block to/from a config file. This argument is mandatory.

  * --conf=(config file path)
> Application and local scope configuration file. This argument is mandatory.

  * --hostname=(hostname)
> A target hostname of the block that will be added. When setting --command=add, this parameter is mandatory.

  * --block=(block\_name)
> A block name of the config file. When adding a block, the block name must not be identical to existing blocks in the config file. When deleting a block, the block name must exist in the config file. If --block is not set, the block name will be "server_$hostname"._

  * --params=(key1=value1;key2=value2;...)
> Parameter lists, separated by semi-colon.