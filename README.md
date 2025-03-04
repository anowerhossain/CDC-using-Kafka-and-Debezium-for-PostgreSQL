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

### Create a Logical Replication Slot

```sql
SELECT * FROM pg_create_logical_replication_slot('debezium_slot', 'pgoutput');
```

### Set Up Debezium PostgreSQL Connector

✅ Kafka (KRaft mode) is already running on Server-1, so configure Debezium.

- Make a separate directory for plugins and navigate in the directory

```bash
 mkdir /usr/local/share/kafka/plugins
 cd /usr/local/share/kafka/plugins
```

- Manually Download & Extract the Correct JARs

```bash
cd /usr/local/share/kafka/plugins/debezium-postgres/
wget https://repo1.maven.org/maven2/io/debezium/debezium-connector-postgres/2.5.0.Final/debezium-connector-postgres-2.5.0.Final-plugin.tar.gz
tar -xvzf debezium-connector-postgres-2.5.0.Final-plugin.tar.gz
rm debezium-connector-postgres-2.5.0.Final-plugin.tar.gz
```
This will extract the required JARs into the correct directory.

- Set Plugin Path in Kafka Connect
```bash
vi kafka/config/connect-distributed.properties
```
Change it to 

```bash
plugin.path=/usr/local/share/kafka/plugins/debezium-postgres/
```

- Stop Kafka Connect if it's running:

```bash
ps aux | grep connect-distributed
kill -9 <PID>  # Replace <PID> with actual process ID
```

- Start the kafka connect again to apply changes

```bash
./bin/connect-distributed.sh config/connect-distributed.properties &
```
Wait a few seconds, then check if Debezium appears.

```bash
curl -X GET http://161.97.143.145:8083/connector-plugins
```
Now, `io.debezium.connector.postgresql.PostgresConnector` should appear.

- Make a separate directory for the connector JSON and navigate in the dorectory

```bash
mkdir /opt/debezium
cd /opt/debezium
```
- Create a JSON file `postgres-connector.json` for connector information.

```bash
touch postgres-connector.json
```

```json
{
  "name": "postgres-source-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "tasks.max": "1",
    "database.hostname": "161.97.143.145",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "debezium123",
    "database.dbname": "postgres",
    "database.server.name": "161.97.143.145",
    "topic.prefix": "server1",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_slot",
    "database.history.kafka.bootstrap.servers": "161.97.143.145:9092",
    "database.history.kafka.topic": "schema-changes.postgres",
    "table.include.list": "public.customers"
  }
}
```

- Load the Connector into Kafka Connect

```bash
curl -X POST -H "Content-Type: application/json" \
  --data @/opt/debezium/postgres-connector.json \
  http://localhost:8083/connectors
```

- Verify the Connector

```bash
curl -X GET http://localhost:8083/connectors
```

It will show

```json
["postgres-source-connector"]
```

### Kafka -JDBC sink connector

- Download the kafka jdbc sink connector from confluent

```bash
wget https://packages.confluent.io/maven/io/confluent/kafka-connect-jdbc/10.6.0/kafka-connect-jdbc-10.6.0.jar
```

- Make a seperate directory inside the plugin directory
```bash
mkdir -p /usr/local/share/kafka/plugins/kafka-connect-jdbc
```

- Move the downloaded .jar file inside the directory we created
```bash
mv kafka-connect-jdbc-10.6.0.jar /usr/local/share/kafka/plugins/kafka-connect-jdbc/
```

- Stopping the current kafka connect process
```bash
ps aux | grep connect-distributed
kill -9 <PID>
```

- Restart the kafka connect process to load the plugin 
```bash
./bin/connect-distributed.sh config/connect-distributed.properties &
```

- To see the connected plugin run
```bash
curl -X GET http://localhost:8083/connector-plugins
```
Look for an entry similar to `io.confluent.connect.jdbc.JdbcSinkConnector` in the returned list.

- Navigate to `/opt/debezium/` and create `postgres-sink-connector.json`
```bash
cd /opt/debezium/
touch postgres-sink-connector.json
```

- Set all the configuration inside the `postgres-sink-connector.json`
```json
{
  "name": "postgres-sink-connector",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "tasks.max": "1",
    "connection.url": "jdbc:postgresql://75.119.ANO.WER:5432/postgres",
    "connection.user": "postgres",
    "connection.password": "your_password",
    "auto.create": "true",
    "auto.evolve": "true",
    "insert.mode": "upsert",
    "pk.fields": "id",
    "pk.mode": "record_value",
    "topics": "server1.public.customers"
  }
}
```
- Run the following command to load the JDBC Sink Connector into Kafka Connect
```bash
curl -X POST -H "Content-Type: application/json" \
  --data @/opt/debezium/postgres-sink-connector.json \
  http://161.97.ANO.WER:8083/connectors
```

- Verify that the connector is running
```bash
curl -X GET http://161.97.ANO.WER:8083/connectors
```

If everything goes fine you will see 
```python
["postgres-source-connector", "postgres-sink-connector"]
```
