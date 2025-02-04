# CDC-using-Kafka-and-Debezium-for-PostgreSQL
We will capture changes (INSERT, UPDATE, DELETE) from PostgreSQL on Server-1 and apply them to PostgreSQL on Server-2 using Debezium and Kafka Connect.

### 🔹 Requirements

✅ Server-1: Runs PostgreSQL (source DB), Kafka, Kafka Connect, and Debezium.

✅ Server-2: Runs PostgreSQL (destination DWH).

✅ Kafka Connect is running on Server-1 using KRaft mode.

### Configure PostgreSQL (Source) for Logical Replication
- Edit PostgreSQL Configuration File

```bash
wal_level = logical
max_replication_slots = 4
max_wal_senders = 4
```

- Restart PostgreSQL

```bash
sudo systemctl restart postgresql
```

### Create a Replication User in PostgreSQL
- Debezium needs a user with replication privileges.

```sql
CREATE USER debezium WITH REPLICATION LOGIN PASSWORD 'debezium123';
ALTER USER debezium WITH SUPERUSER;
```

### Create a Table for CDC in PostgreSQL (Server-1)

```sql
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```
Enable replication for this table:

```sql
ALTER TABLE customers REPLICA IDENTITY FULL;
```

