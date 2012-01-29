---
layout: post
title: PostgreSQL Recovery
---

We ran out of disk space on a database server, crashing PostgreSQL as a
result. Unfortunately, PostgreSQL was unable to recover itself upon
restart:

    root@host# /etc/init.d/postgresql-8.3 start
     * Starting PostgreSQL 8.3 database server
     * The PostgreSQL server failed to start. Please check the log output:
    2010-08-01 08:21:19 EDT LOG:  database system was interrupted; last known up at 2010-07-31 23:00:05 EDT
    2010-08-01 08:21:19 EDT FATAL:  invalid data in file "00000001000002BD00000072.00000020.backup"
    2010-08-01 08:21:19 EDT LOG:  startup process (PID 9203) exited with exit code 1
    2010-08-01 08:21:19 EDT LOG:  aborting startup due to startup process failure

`00000001000002BD00000072.00000020.backup` is a zero-length file in the
`pg_xlog` directory. We tried deleting that file, but then the restart
threw a different error:

    root@host# /etc/init.d/postgresql-8.3 start
     * Starting PostgreSQL 8.3 database server
     * The PostgreSQL server failed to start. Please check the log output:
    2010-08-01 08:26:11 EDT LOG:  database system was interrupted; last known up at 2010-07-31 23:00:05 EDT
    2010-08-01 08:26:11 EDT LOG:  could not open file "pg_xlog/00000001000002BD00000072" (log file 701, segment 114): No such file or directory
    2010-08-01 08:26:11 EDT LOG:  invalid checkpoint record
    2010-08-01 08:26:11 EDT PANIC:  could not locate required checkpoint record
    2010-08-01 08:26:11 EDT HINT:  If you are not restoring from a backup, try removing the file "/var/lib/postgresql/8.3/main/backup_label".
    2010-08-01 08:26:11 EDT LOG:  startup process (PID 9419) was terminated by signal 6: Aborted
    2010-08-01 08:26:11 EDT LOG:  aborting startup due to startup process failure

We concluded that a point-in-time recovery (PITR) was required. First,
we saved our current cluster data directory. (In retrospect, we needed
the unarchived WAL files from the `pg_xlog` directory when restoring, so
we should have copied them off to the side first.)

    cd /var/lib/postgresql/8.3/main/
    tar zcvf /var/backups/postgresql/8.3/main/snapshot/2010-08-01-CRASHED.tar.gz .

Next, we restored our last known good snapshot. The one from the morning
of the crash looked suspiciously small, so we went back to the previous
day:

    rm -rf  *
    tar jxvf /var/backups/postgresql/8.3/main/snapshot/2010-07-30.tar.bz2 
    mkdir -p pg_xlog/archive_status
    chown -R postgres.postgres pg_xlog
    chmod -R 700 pg_xlog

Now we copied the unarchived WAL files from before the recovery into the
`pg_xlog` directory. We then created `/var/lib/postgresql/8.3/main/recovery.conf` as
follows:

    restore_command = 'bzcat /var/backups/postgresql/8.3/main/wal/%f.bz2 > %p'
    recovery_target_time = '2010-07-31 06:28:00 EDT'


We also made `recovery.conf` owned by the `postgres` user.

Finally, we started PostgreSQL via `/etc/init.d/postgresql-8.3 start`.


