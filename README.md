# ctfd-hsr

CTFd setup for HSR

## CTFd configuration

Fill the [.env](.env) and [ctfd-app.env](ctfd-app.env) files.

```bash
docker compose up -d
docker compose logs -f
```

## mysqld-exporter

```bash
docker exec -it ctfd-hsr-db-1 mysql -u root -p
```

```sql
CREATE USER 'exporter'@'%' IDENTIFIED BY 'XXXXXXXX' WITH MAX_USER_CONNECTIONS 3;
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';
```

Adapt the [mysqld-exporter/my.cnf](mysqld-exporter/my.cnf) file.

## ctfd-exporter

Fill the [ctfd-exporter.env](ctfd-exporter.env) file.

```env
# API key for the CTFd instance
CTFD_API=<key>
```
