
# References about how MHA works #
(Disclaimer: This page covers internal design notes. This page is not as structured as other pages, and might not be up to date)

  * [Slides at the MySQL Conference and Expo 2011](http://www.slideshare.net/matsunobu/automated-master-failover)

Most of basic algorithms are covered in the slides, but I have changed or added a bit after publishing the slides.

# How to identify from which relay log position needs to be applied #
(slides p.17-19, 29)

Long time ago (before publishing) MHA used mysqlbinlog to identify starting relay log position, but now MHA doesn't use mysqlbinlog for that purpose. Instead, MHA parses binary log headers to identify the target position.

Binary log format is published at [MySQL Forge](http://forge.mysql.com/wiki/MySQL_Internals_Binary_Log), and parsing binlog header is not so difficult.

The reason why MHA parses binlog header by itself instead of using mysqlbinlog is that currently mysqlbinlog has some issues for that purpose. The biggest reason is that mysqlbinlog does not have an option to print binlog header only, but it always prints binlog body (SQL statements, etc).

To identify the target relay log position, all necessary information(event type, server id, event length, end log pos) is stored in binlog header so parsing binlog body is not needed. mysqlbinlog not only adds performance overheads for reading binlog data, but also we need to care about some types of SQL statements.
As described [in the p.29 of the slides](http://www.slideshare.net/matsunobu/automated-master-failover/29), statement based binary log may contain position information(# at or end\_log\_pos) within SQL statements.

To avoid false recognition of the position information, using mysqlbinlog --base64-output=always is helpful, as described in the slide. But mysqlbinlog --base64-output=always is supported in MySQL 5.1 and 5.5 only. It's not supported in 5.0, and it's removed in 5.6. I don't like DBAs have to use very limited number of mysqlbinlog versions.

## Fast relay log position search ##

Suppose target master log file:pos is mysqld-bin.000001:504810023, latest master log file:pos is mysqld-bin.000001:504810689, and the latest relay log file is mysqld-relay-bin.000001 and file size is more than 500MB. Position difference is only 666 bytes. If MHA parses the relay log from the beginning, it has to parse more than 500MB, which will take long time. This situation is highly expected, and taking long time to recover causes longer downtime.

MHA does not always scan the relay log file from the beginning, but scans from the differential position, based on the following formula.

```
  $latest_mlf = Master_Log_File on the latest slave
  $target_mlf = Master_Log_File on the recovery target slave
  $latest_rmlp = Read_Master_Log_Pos on the latest slave 
  $target_rmlp = Read_Master_Log_Pos on the recovery target slave
  $offset = $latest_rmlp - $target_rmlp
  $filesize = File size of the latest relay log file on the latest slave

  if ($latest_mlf eq $target_mlf) && ($filesize > $offset), and if binlog events can be readable from ($filesize - $offset) position, MHA decides that starting recovery position is from ($filesize - $offset) position from the latest relay log file on the latest slave.
```

This can avoid scanning large relay log file, so this is very fast.


# How to generate SQL statements for applying to target slaves #

Originally I designed as below (this was described in the MySQL Conference slides).

  1. Concatenating mysqlbinlog output from exec pos to read pos of the target slave, trimming implicit ROLLBACK commands
  1. Concatenating mysqlbinlog output from read pos of the target slave to read pos of the latest slave, trimming implicit ROLLBACK commands
  1. Concatenating mysqlbinlog output from read pos of the latest slave to the master's tail of the binary log, trimming implicit ROLLBACK commands
  1. Apply the output to the target slave by mysql command

This algorithm had some serious issues. Now MHA works as below.

  1. Concatenating binary logs from exec pos to read pos of the target slave (not mysqlbinlog output, but raw binlog)
  1. Concatenating binary logs from read pos of the target slave to read pos of the latest slave (not mysqlbinlog output, and trimming format description event)
  1. Concatenating binary logs from read pos of the latest slave to the master's tail of the binary log (not mysqlbinlog output, and trimming format description event)
  1. Executing mysqlbinlog for the concatenated binlog, then apply it to the target slave by mysql command

The original option can still be used by setting "handle\_row\_binlog=0" in /etc/masterha\_default.cnf, but it's deprecated.

The reason of the change is the following.

  * To support row based binary log events

Currently mysqlbinlog does not print valid SQL (BINLOG) statements correctly unless the whole row based events are written in the single binary log file. For example, if partial events (TABLE\_MAP\_EVENT event) are written in mysqld-relay-bin.000001 and the rest of the events (WRITE\_ROWS events) are written in mysqld-relay-bin.000002, mysqlbinlog does not print valid events for the row events. So the original approach (concatenating multiple mysqlbinlog outputs) doesn't always fork for row based format. The new approach works for both row based and statement based format because mysqlbinlog target becomes single binlog file.

Related bug reports: http://bugs.mysql.com/bug.php?id=60964


  * To make mysqlbinlog filtering not needed

mysqlbinlog implicitly adds ROLLBACK statements (and equivalent BINLOG events) at the end of the mysqlbinlog output and in some cases at the beginning of the each binlog file. Such ROLLBACK statements need to be trimmed in order to avoid partial transaction rollback. Parsing all mysqlbinlog outputs and replacing the target ROLLBACK and equivalent BINLOG statements will make it work, but parsing all output adds performance overhead. The new approach will result in single binlog file so ROLLBACK statements are never added in the middle of the transaction.

So, MHA creates a single large binlog file to apply to each target slave. When it needs concatenate multiple binlog/relay log files, MHA trims format description events from except first binlog file. The reason is that format description event might generate unnecessary ROLLBACK statements from mysqlbinlog.