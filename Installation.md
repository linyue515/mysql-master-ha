
# Installation #
> MHA consists of MHA Manager and MHA Node packages. MHA Manager runs on a manager server, and MHA Node runs on each MySQL server. MHA Node programs do not run always, but are invoked from MHA manager programs when needed (at configuration check, failover, etc).
> Both MHA Manager and MHA Node are written in Perl.

# Downloading MHA Node and MHA Manager #
> MHA Node and MHA Manager can be downloaded [from "Downloads" section](http://code.google.com/p/mysql-master-ha/downloads/list). These are stable packages.

> If you want to try development source trees, check out GitHub source trees. MHA Manager is hosted [here](http://github.com/yoshinorim/mha4mysql-manager), and MHA Node is hosted [here](https://github.com/yoshinorim/mha4mysql-node).

# Installing MHA Node #
> MHA Node has scripts and dependent perl modules that do the following.

  * save\_binary\_logs: Saving and copying dead master's binary logs
  * apply\_diff\_relay\_logs: Identifying differential relay log events and applying all necessary log events
  * purge\_relay\_logs: Purging relay log files

> You need to install MHA Node to all MySQL servers (both master and slave). You also need to install MHA Node on a management server because MHA Manager modules internally depend on MHA Node modules.
> MHA Manager internally connects to managed MySQL servers via SSH and executes MHA Node scripts.
> MHA Node does not depend on any external Perl modules except DBD::mysql so you should be able to install easily.

> On RHEL/CentOS distribution, you can install MHA Node rpm package as below.

```
  ## If you have not installed DBD::mysql, install it like below, or install from source.
  # yum install perl-DBD-MySQL

  ## Get MHA Node rpm package from "Downloads" section.
  # rpm -ivh mha4mysql-node-X.Y-0.noarch.rpm
```

> On Ubuntu/Debian distribution, you can install MHA Node deb package as below.

```
  ## If you have not installed DBD::mysql, install it like below, or install from source.
  # apt-get install libdbd-mysql-perl

  ## Get MHA Node deb package from "Downloads" section.
  # dpkg -i mha4mysql-node_X.Y_all.deb
```

> You can also install MHA Node from source.

```
  ## Install DBD::mysql if not installed
  $ tar -zxf mha4mysql-node-X.Y.tar.gz
  $ perl Makefile.PL
  $ make
  $ sudo make install
```


# Installing MHA Manager #
> MHA Manager has administrative command line programs such as masterha\_manager, masterha\_master\_switch, etc, and dependent Perl modules.
> MHA Manager depends on the following Perl modules. You need to install them before installing MHA Manager. Do not forget to install MHA Node.

  * MHA Node package
  * DBD::mysql
  * Config::Tiny
  * Log::Dispatch
  * Parallel::ForkManager
  * Time::HiRes (included from Perl v5.7.3)

> On RHEL/CentOS distribution, you can install MHA Manager rpm package as below.

```
  ## Install dependent Perl modules
  # yum install perl-DBD-MySQL
  # yum install perl-Config-Tiny
  # yum install perl-Log-Dispatch
  # yum install perl-Parallel-ForkManager

  ## Install MHA Node, since MHA Manager uses some modules provided by MHA Node.
  # rpm -ivh mha4mysql-node-X.Y-0.noarch.rpm

  ## Finally you can install MHA Manager
  # rpm -ivh mha4mysql-manager-X.Y-0.noarch.rpm
```


> On Ubuntu/Debian distribution, you can install MHA Manager deb package as below.

```
  ## Install dependent Perl modules
  # apt-get install libdbd-mysql-perl
  # apt-get install libconfig-tiny-perl
  # apt-get install liblog-dispatch-perl
  # apt-get install libparallel-forkmanager-perl

  ## Install MHA Node, since MHA Manager uses some modules provided by MHA Node.
  # dpkg -i mha4mysql-node_X.Y_all.deb

  ## Finally you can install MHA Manager
  # dpkg -i mha4mysql-manager_X.Y_all.deb
```


> You can also install MHA Manager from source.

```
  ## Install dependent Perl modules
  # MHA Node (See above)
  # Config::Tiny
  ## perl -MCPAN -e "install Config::Tiny"
  # Log::Dispatch
  ## perl -MCPAN -e "install Log::Dispatch"
  # Parallel::ForkManager 
  ## perl -MCPAN -e "install Parallel::ForkManager"
  ## Installing MHA Manager
  $ tar -zxf mha4mysql-manager-X.Y.tar.gz
  $ perl Makefile.PL
  $ make
  $ sudo make install
```