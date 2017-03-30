#DBAL Errata
Documentation section lists non obvious behaviours of DBAL component and ways to avoid them.

## Timestamps
Consider using 'datetime' type for your columns instead of 'timestamp' as it gives you more unified support and ability to define proper default values.

> Timestamp fields behaviour is different in MySQL 5.5 and MySQL 5.6+!