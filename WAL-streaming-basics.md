# Configuration

[Install Postgres](https://www.postgresql.org/download/)

Create a new DB:
```bash
initdb -A trust -D ./db1
```

Change postgres.conf:
```
#------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

# - Connection Settings -

listen_addresses = '*' <-- listen all addresses  
										# what IP address(es) to listen on;
                                        # comma-separated list of addresses;
                                        # defaults to 'localhost'; use '*' for all
                                        # (change requires restart)
port = 5400 # <--- change port          # (change requires restart)
```

Start a brand new master:
```bash
pg_ctl -D ./db1 -l db1_log start
```

Connect to server:
```bash
pgcli -p 5400 postgres
```

Create replication user:
```sql
CREATE USER repuser REPLICATION;
```

*(not necessary if you have turned off auth)* Edit pg_hba.conf:
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             repuser         127.0.0.1/32            trust # <----
host    all             all             127.0.0.1/32            trust
# IPv6 local connections:
host    all             all             ::1/128                 trust
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     trust
host    replication     all             127.0.0.1/32            trust
```

Restart master:
```bash
pg_ctl -D db1/ -l logfile restart
```

Create copy of a master
```bash
pg_basebackup -h localhost -U repuser --checkpoint=fast \
-D ./db2 -R --slot=slot_name -C --port=5400
```
**-h** localhost - hostname

**-U** repuser - create backup using replication user

**--checkpoint**=fast - sets checkpoint mode to fast

**-D** ./db2 - replica db location

**-R** - configure replication

**--slot**=slot_name - name replication slot on master

**-C** - create replication slot on master

**--port**=5400 - db port

file named `standby.signal` tells pg that this is replica
```ls
total 384
drwx------@ 27 noodleheaded  staff     864 May 28 11:53 .
drwxr-xr-x@  5 noodleheaded  staff     160 May 28 11:53 ..
-rw-------@  1 noodleheaded  staff     225 May 28 11:53 backup_label
-rw-------@  1 noodleheaded  staff  137924 May 28 11:53 backup_manifest
drwx------@  5 noodleheaded  staff     160 May 28 11:53 base
drwx------@ 62 noodleheaded  staff    1984 May 28 11:53 global
drwx------@  2 noodleheaded  staff      64 May 28 11:53 pg_commit_ts
drwx------@  2 noodleheaded  staff      64 May 28 11:53 pg_dynshmem
-rw-------@  1 noodleheaded  staff    5760 May 28 11:53 pg_hba.conf
-rw-------@  1 noodleheaded  staff    2640 May 28 11:53 pg_ident.conf
drwx------@  5 noodleheaded  staff     160 May 28 11:53 pg_logical
drwx------@  4 noodleheaded  staff     128 May 28 11:53 pg_multixact
drwx------@  2 noodleheaded  staff      64 May 28 11:53 pg_notify
drwx------@  2 noodleheaded  staff      64 May 28 11:53 pg_replslot
drwx------@  2 noodleheaded  staff      64 May 28 11:53 pg_serial
drwx------@  2 noodleheaded  staff      64 May 28 11:53 pg_snapshots
drwx------@  2 noodleheaded  staff      64 May 28 11:53 pg_stat
drwx------@  2 noodleheaded  staff      64 May 28 11:53 pg_stat_tmp
drwx------@  2 noodleheaded  staff      64 May 28 11:53 pg_subtrans
drwx------@  2 noodleheaded  staff      64 May 28 11:53 pg_tblspc
drwx------@  2 noodleheaded  staff      64 May 28 11:53 pg_twophase
-rw-------@  1 noodleheaded  staff       3 May 28 11:53 PG_VERSION
drwx------@  5 noodleheaded  staff     160 May 28 11:53 pg_wal
drwx------@  3 noodleheaded  staff      96 May 28 11:53 pg_xact
-rw-------@  1 noodleheaded  staff     466 May 28 11:53 postgresql.auto.conf
-rw-------@  1 noodleheaded  staff   32415 May 28 11:53 postgresql.conf
-rw-------@  1 noodleheaded  staff       0 May 28 11:53 standby.signal # <----
```

Configure postgres.conf on replica:
```
#------------------------------------------------------------------------------
# CONNECTIONS AND AUTHENTICATION
#------------------------------------------------------------------------------

# - Connection Settings -

listen_addresses = '*'                  # what IP address(es) to listen on;
                                        # comma-separated list of addresses;
                                        # defaults to 'localhost'; use '*' for all
                                        # (change requires restart)
port = 5401                             # (change requires restart)
```

Start replica:
```bash
pg_ctl -D db2 -l logfile2 start
```

Hooray, we have started streaming replication
```
2025-05-28 11:56:45.813 MSK [80366] LOG:  listening on IPv4 address "0.0.0.0", port 5401
2025-05-28 11:56:45.813 MSK [80366] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5401"
2025-05-28 11:56:45.818 MSK [80372] LOG:  database system was interrupted; last known up at 2025-05-28 11:53:07 MSK
2025-05-28 11:56:45.851 MSK [80372] LOG:  starting backup recovery with redo LSN 0/2000028, checkpoint LSN 0/2000080, on timeline ID 1
2025-05-28 11:56:45.852 MSK [80372] LOG:  entering standby mode
2025-05-28 11:56:45.855 MSK [80372] LOG:  redo starts at 0/2000028
2025-05-28 11:56:45.855 MSK [80372] LOG:  completed backup recovery with redo LSN 0/2000028 and end LSN 0/2000158
2025-05-28 11:56:45.855 MSK [80372] LOG:  consistent recovery state reached at 0/2000158
2025-05-28 11:56:45.855 MSK [80366] LOG:  database system is ready to accept read-only connections
2025-05-28 11:56:45.889 MSK [80373] LOG:  started streaming WAL from primary at 0/3000000 on timeline 1
``` 

Connect to replica
```
pgcli -p 5401 postgres
```

Create data on master
```sql
CREATE TABLE users(id int);
INSERT INTO users VALUES (1);
INSERT INTO users VALUES (2);
```

Get data from replica
```sql
SELECT * FROM users;
```
