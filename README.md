# MySQL Master-Slave Replication Setup

A production-tested guide for configuring MySQL master-slave replication on Ubuntu Server, based on a real deployment at a 300-bed public hospital managing live clinical and scheduling data.

---

## Background

At Garki Hospital Abuja, a single MySQL instance held all patient scheduling and clinical records. One hardware failure would have meant complete data loss with no recovery path. This setup was designed and deployed to solve that problem — replicating the production database (`medic_prod2`) to a dedicated replica server, with automated monitoring and a tested failover runbook.

**Outcome:** Zero unplanned database outages over 18 months post-deployment. Recovery time objective (RTO) dropped from several hours to under 5 minutes.

---

## Architecture

```
Master Server (10.1.10.249)          Slave/Replica Server (10.1.10.211)
┌─────────────────────────┐          ┌─────────────────────────────┐
│  MySQL (medic_prod2)    │──────────│  MySQL (medic_prod2 replica) │
│  server-id = 1          │  Port    │  server-id = 2               │
│  Binary logging ON      │  3306    │  Replicates from master       │
└─────────────────────────┘          └─────────────────────────────┘
```

---

## Prerequisites

- Two Ubuntu servers with network access between them
- MySQL Server and MySQL Client installed on both
- Root access on both servers
- Note the IPs for your environment (this guide uses `10.1.10.249` for master, `10.1.10.211` for slave)

---

## Part 1: Master Server Setup

### Step 1 — Update and install MySQL

```bash
sudo apt-get update
sudo apt-get install mysql-server mysql-client -y
```

### Step 2 — Allow the slave server to connect on port 3306

Replace `10.1.10.211` with your slave server's IP address.

```bash
sudo ufw allow from 10.1.10.211 to any port 3306
```

Expected output: `Rule added`

### Step 3 — Configure the master database

Open the MySQL configuration file:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Make the following changes (replace the IP and database name to match your environment):

```ini
bind-address        = 10.1.10.249
server-id           = 1
log_bin             = /var/log/mysql/mysql-bin.log
binlog_do_db        = medic_prod2
```

Also uncomment `expire_logs_days` and `max_binlog_size` if they are commented out.

Save and close: `CTRL + X`, then `Y`, then `ENTER`

### Step 4 — Restart MySQL

```bash
sudo service mysql restart
```

### Step 5 — Create a dedicated replication user

Open the MySQL shell:

```bash
mysql -u root -p
```

Run these three commands:

```sql
CREATE USER 'slave_user'@'10.1.10.211' IDENTIFIED BY 'your_secure_password';
GRANT REPLICATION SLAVE ON *.* TO 'slave_user'@'10.1.10.211';
FLUSH PRIVILEGES;
```

### Step 6 — Lock tables and capture binary log position

Select the target database and lock tables to prevent writes during the snapshot:

```sql
RESET MASTER;
USE medic_prod2;
FLUSH TABLES WITH READ LOCK;
```

### Step 7 — Record the binary log coordinates

**Open a second terminal window** and run:

```bash
mysql -u root -p
```

```sql
SHOW MASTER STATUS\G;
QUIT;
```

**Important:** Copy the `File` and `Position` values from the output. You will need these in Part 2, Step 23. Take a screenshot if needed — these coordinates must match exactly.

Example output:
```
File: mysql-bin.000001
Position: 154
```

### Step 8 — Create a database dump

Still in the second terminal (outside the MySQL shell):

```bash
mysqldump -u root -p medic_prod2 > medic_prod2.sql
```

### Step 9 — Unlock the tables

Return to the first terminal and unlock:

```sql
UNLOCK TABLES;
QUIT;
```

### Step 10 — Copy the dump to the slave server

```bash
scp medic_prod2.sql administrator@10.1.10.211:/home/administrator/Documents
```

> **If you get an SSH authentication error**, run the following on the master first, then retry:
> ```bash
> ssh-keygen -R 10.1.10.211
> ```

---

## Part 2: Slave/Replica Server Setup

### Step 11 — Update and install MySQL

```bash
sudo apt-get update
sudo apt-get install mysql-server mysql-client -y
```

### Step 12 — Configure the slave database

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Set the following (note `server-id = 2` to distinguish it from the master):

```ini
bind-address        = 10.1.10.211
server-id           = 2
log_bin             = /var/log/mysql/mysql-bin.log
binlog_do_db        = medic_prod2
```

Uncomment `expire_logs_days` and `max_binlog_size`. Save and close.

### Step 13 — Restart MySQL

```bash
sudo service mysql restart
```

### Step 14 — Prepare the slave database

```bash
mysql -u root -p
```

```sql
STOP SLAVE;
RESET SLAVE;
CREATE DATABASE medic_prod2;
```

### Step 15 — Import the dump file

**Open a second terminal on the slave server:**

```bash
cd /home/administrator/Documents
mysql -u root -p medic_prod2 < medic_prod2.sql
```

> **If you see a duplicate entry error**, drop the database and start again from Step 6 (master side):
> ```sql
> DROP DATABASE medic_prod2;
> ```

### Step 16 — Configure replication and start the slave

Return to the first slave terminal. Use the `File` and `Position` values you recorded in Step 7:

```sql
STOP SLAVE;
CHANGE MASTER TO
  MASTER_HOST='10.1.10.249',
  MASTER_USER='slave_user',
  MASTER_PASSWORD='your_secure_password',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=154;
START SLAVE;
```

### Step 17 — Verify replication is running

```sql
SHOW SLAVE STATUS\G;
```

A successful setup will show:

```
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
```

If both values are `Yes`, replication is active. Any writes to `medic_prod2` on the master will now replicate to the slave automatically.

---

## Troubleshooting

### Error 1594 (relay log read failure)

Run the following on the slave, substituting `Relay_Master_Log_File` and `Exec_Master_Log_Pos` values from `SHOW SLAVE STATUS\G`:

```sql
STOP SLAVE;
RESET SLAVE;
CHANGE MASTER TO
  MASTER_LOG_FILE='mysql-bin.XXXXXX',
  MASTER_LOG_POS=XXXXXXX;
START SLAVE;
SHOW SLAVE STATUS\G;
```

### Duplicate entry errors (Error 1062)

If duplicate errors are preventing the slave from syncing, configure the slave to skip them:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Add under `[mysqld]`:

```ini
slave-skip-errors = 1062
skip-slave-start
```

Save, then restart:

```bash
sudo service mysql restart
```

Then repeat Step 16.

---

## Results

| Metric | Before | After |
|--------|--------|-------|
| Single point of failure | Yes | No |
| Recovery time (data loss event) | Hours (manual restore) | Under 5 minutes |
| Unplanned outages (18 months post-deploy) | N/A | 0 |
| Data continuity for clinical records | At risk | 100% |

---

## Notes

- Tested on Ubuntu Server 20.04 with MySQL 8.0
- Production database: hospital EMR System (~medic_prod2)
- Monitoring was configured separately using automated health checks against `SHOW SLAVE STATUS`
- A full failover runbook was maintained alongside this setup for operational handoff
