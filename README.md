# PostgreSQL Hands-On Lab: Backups, PITR, and Replication

## Objective
- Create and verify a logical backup
- Configure WAL archiving and take a base backup
- Perform point-in-time recovery (PITR)
- Set up and monitor a streaming standby replica

## 1. Take and Verify a Logical Backup

```bash
mkdir -p ~/backups

pg_dump -Fc -f ~/backups/bootcamp.dump bootcamp
pg_restore --list ~/backups/bootcamp.dump | head

createdb bootcamp_check
pg_restore -d bootcamp_check ~/backups/bootcamp.dump

psql -d bootcamp_check -c "SELECT count(*) FROM students;"
```

## 2. Enable WAL Archiving

Add to `postgresql.conf`:

```conf
wal_level = replica
archive_mode = on
archive_command = 'cp %p /home/$USER/backups/wal/%f'
```

Run:

```bash
mkdir -p ~/backups/wal
sudo systemctl restart postgresql

pg_basebackup -D ~/backups/base -Fp -Xs -P
```

## 3. Simulate Disaster and Recover

Record the recovery time:

```sql
SELECT now();
```

Simulate accidental deletion:

```sql
DELETE FROM students;
```

Stop PostgreSQL:

```bash
sudo systemctl stop postgresql
```

Restore the base backup to the PostgreSQL data directory, then add to `postgresql.conf`:

```conf
restore_command = 'cp ~/backups/wal/%f %p'
recovery_target_time = '2026-10-10 21:08:00.123456+03'
recovery_target_action = 'promote'
```

Create the recovery signal file:

```bash
touch ~/backups/base/recovery.signal
```

Start PostgreSQL:

```bash
sudo systemctl start postgresql
```

Verify recovery:

```sql
SELECT count(*) FROM students;
SELECT pg_is_in_recovery();
```

## 4. Set Up Streaming Replication

On the primary:

```sql
CREATE ROLE replicator
WITH REPLICATION LOGIN PASSWORD 'reppass';
```

Add to `pg_hba.conf`:

```conf
host replication replicator 127.0.0.1/32 md5
```

Reload PostgreSQL:

```bash
sudo systemctl reload postgresql
```

Create the standby:

```bash
pg_basebackup -h 127.0.0.1 -U replicator -D ~/standby -R -P
```

Start the standby on a separate port if it is on the same machine:

```bash
pg_ctl -D ~/standby -o "-p 5433" -l ~/standby/standby.log start
```

## 5. Monitor Replication

On the primary:

```sql
SELECT application_name,
       state,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

Test replication:

```sql
INSERT INTO students (name) VALUES ('replication_test');
```

On the standby:

```sql
SELECT * FROM students WHERE name = 'replication_test';
```
