docker exec -it pg_master bash
mkdir -p /var/lib/postgresql/data/backups
pg_basebackup -h localhost -U replicator -D /var/lib/postgresql/data/backups/pg_backup -v -P --wal-method=stream
