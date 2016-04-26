# Monitoring multiple applications #
> You may want to monitor multiple (master, slaves) pairs from single manager server. This is easy. Just create a new configuration file for application 2 and start masterha\_manager by setting --conf=(another configuration file).

```
  # masterha_manager --conf=/etc/conf/masterha/app1.cnf
  # masterha_manager --conf=/etc/conf/masterha/app2.cnf
```

> If you have some identical parameters between app1 and app2, setting these parameters at global configuration file is recommended. See [Configuration](Configuration.md) section for details.

