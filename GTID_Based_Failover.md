# GTID based failover #
> Starting from MHA 0.56, MHA supported both GTID based failover and traditional relay log based failover. MHA automatically distinguishes which failover to choose. To do GTID based failover, all of the following is needed.

  * Using MySQL 5.6 (or later)
  * All MySQL instances use gtid\_mode=1
  * At least one of the instances has Auto\_Position enabled