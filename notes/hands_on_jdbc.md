# Hands-On: Kafka Connect with JDBC
**Local Setup - Confluent Platform + PostgreSQL**

## Goal

```
PostgreSQL (source DB)
    -> JDBC Source Connector
        -> Kafka (with Schema Registry)
            -> JDBC Sink Connector
                -> PostgreSQL (sink DB)
```

**We will:**
1. Set up two PostgreSQL databases (source + sink)
2. Start Kafka, Schema Registry, and Kafka Connect
3. Create a JDBC Source Connector (reads from source DB)
4. Create a JDBC Sink Connector (writes to sink DB)
5. Observe data flowing end-to-end
6. Explore Schema Registry

---

## Step 1: Set Up PostgreSQL Databases

Connect to PostgreSQL:
```bash
psql -U postgres
```

Create source and sink databases:
```sql
CREATE DATABASE connect_source;
CREATE DATABASE connect_sink;
```

Connect to source database and create table with sample data:
```sql
\c connect_source

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO users (name, email) VALUES
    ('Alice', 'alice@example.com'),
    ('Bob', 'bob@example.com'),
    ('Charlie', 'charlie@example.com');

SELECT * FROM users;

\q
```

---

## Step 2: Start Kafka, Schema Registry, Kafka Connect

> Start each in a **SEPARATE terminal**. Wait for each to fully start before starting the next one.

### Terminal 1: Start Kafka

```bash
# Using confluent CLI
confluent local services kafka start

# OR manually (KRaft):
kafka-storage format -t $(kafka-storage random-uuid) \
    -c $CONFLUENT_HOME/etc/kafka/kraft/server.properties

kafka-server-start $CONFLUENT_HOME/etc/kafka/kraft/server.properties
```

### Terminal 2: Start Schema Registry

```bash
confluent local services schema-registry start

# OR manually:
schema-registry-start $CONFLUENT_HOME/etc/schema-registry/schema-registry.properties
```

Verify:
```bash
curl http://localhost:8081/subjects
# Expected: []
```

### Terminal 3: Start Kafka Connect (Standalone)

```bash
connect-standalone \
    $CONFLUENT_HOME/etc/schema-registry/connect-avro-standalone.properties
```

> This worker config uses AvroConverter + Schema Registry by default:
> - `key.converter=io.confluent.connect.avro.AvroConverter`
> - `key.converter.schema.registry.url=http://localhost:8081`
> - `value.converter=io.confluent.connect.avro.AvroConverter`
> - `value.converter.schema.registry.url=http://localhost:8081`
> - `offset.storage.file.filename=/tmp/connect.offsets`

Wait until you see: **"Kafka Connect started"**

Verify:
```bash
# Connect info
curl http://localhost:8083/

# Check available connector plugins
curl http://localhost:8083/connector-plugins | python3 -m json.tool
# Look for: JdbcSourceConnector and JdbcSinkConnector
```

---

## Step 3: Create JDBC Source Connector

This connector reads from `connect_source.users` and writes to a Kafka topic.

```bash
curl -X POST http://localhost:8083/connectors \
    -H "Content-Type: application/json" \
    -d '{
        "name": "jdbc-source-users",
        "config": {
            "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
            "connection.url": "jdbc:postgresql://localhost:5432/connect_source",
            "connection.user": "postgres",
            "connection.password": "postgres",
            "mode": "incrementing",
            "incrementing.column.name": "id",
            "topic.prefix": "pg-",
            "table.whitelist": "users",
            "poll.interval.ms": "5000"
        }
    }'
```

**What this does:**
- Reads the `users` table from `connect_source` database
- Creates Kafka topic `pg-users` (topic.prefix + table name)
- Uses `incrementing` mode: tracks the `id` column
- Polls every 5 seconds for new rows
- Serializes data as Avro (from worker config) and registers schema

### Verify

```bash
# Check connector status
curl http://localhost:8083/connectors/jdbc-source-users/status | python3 -m json.tool
# state should be "RUNNING" for both connector and task

# List topics - should see pg-users
kafka-topics --list --bootstrap-server localhost:9092

# Consume messages (Avro format)
kafka-avro-console-consumer \
    --bootstrap-server localhost:9092 \
    --topic pg-users \
    --from-beginning \
    --property schema.registry.url=http://localhost:8081
```

Expected output (3 records):
```json
{"id":1,"name":"Alice","email":"alice@example.com","created_at":...}
{"id":2,"name":"Bob","email":"bob@example.com","created_at":...}
{"id":3,"name":"Charlie","email":"charlie@example.com","created_at":...}
```

---

## Step 4: Test Real-Time Source Ingestion

In another terminal, insert new rows into the source database:

```bash
psql -U postgres -d connect_source -c \
    "INSERT INTO users (name, email) VALUES ('Dave', 'dave@example.com');"

psql -U postgres -d connect_source -c \
    "INSERT INTO users (name, email) VALUES ('Eve', 'eve@example.com');"
```

Watch the avro console consumer terminal - within 5 seconds (poll interval), the new records should appear:
```json
{"id":4,"name":"Dave","email":"dave@example.com","created_at":...}
{"id":5,"name":"Eve","email":"eve@example.com","created_at":...}
```

> **Note:** `incrementing` mode only catches **NEW** rows (higher ID). It does NOT capture UPDATEs or DELETEs. For that, use `timestamp+incrementing` mode.

---

## Step 5: Create JDBC Sink Connector

This connector reads from Kafka topic `pg-users` and writes to the sink DB.

```bash
curl -X POST http://localhost:8083/connectors \
    -H "Content-Type: application/json" \
    -d '{
        "name": "jdbc-sink-users",
        "config": {
            "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
            "connection.url": "jdbc:postgresql://localhost:5432/connect_sink",
            "connection.user": "postgres",
            "connection.password": "postgres",
            "topics": "pg-users",
            "auto.create": "true",
            "auto.evolve": "true",
            "insert.mode": "upsert",
            "pk.mode": "record_value",
            "pk.fields": "id"
        }
    }'
```

**What this does:**
- Reads from Kafka topic `pg-users`
- Writes to `connect_sink` database
- `auto.create=true`: creates the `pg-users` table automatically
- `auto.evolve=true`: adds new columns if schema changes
- `insert.mode=upsert`: inserts new rows, updates existing ones
- `pk.mode=record_value`: uses `id` field from record as primary key

### Verify

```bash
# Check connector status
curl http://localhost:8083/connectors/jdbc-sink-users/status | python3 -m json.tool

# Check the sink database - data should be there!
psql -U postgres -d connect_sink -c "SELECT * FROM \"pg-users\";"
# Expected: All 5 rows (Alice, Bob, Charlie, Dave, Eve)
```

> **Note:** Table name is `pg-users` (from the topic name), use quotes in SQL.

---

## Step 6: Explore Schema Registry

The source connector automatically registered Avro schemas in Schema Registry.

```bash
# List all subjects
curl http://localhost:8081/subjects
# Expected: ["pg-users-key","pg-users-value"]

# Get the value schema (describes the users table structure)
curl http://localhost:8081/subjects/pg-users-value/versions/latest | python3 -m json.tool
```

The schema will look like:
```json
{
    "type": "record",
    "name": "users",
    "fields": [
        {"name": "id", "type": "int"},
        {"name": "name", "type": "string"},
        {"name": "email", "type": "string"},
        {"name": "created_at", "type": ["null", "long"]}
    ]
}
```

```bash
# Get schema by ID
curl http://localhost:8081/schemas/ids/1
```

### Schema Evolution Test

```bash
# Add a new column to the source table
psql -U postgres -d connect_source -c \
    "ALTER TABLE users ADD COLUMN age INTEGER DEFAULT 25;"

# Insert a row with the new column
psql -U postgres -d connect_source -c \
    "INSERT INTO users (name, email, age) VALUES ('Frank', 'frank@example.com', 30);"

# Check Schema Registry - new version should be registered
curl http://localhost:8081/subjects/pg-users-value/versions
# Expected: [1, 2]  (two versions now)

# Compare version 1 vs version 2
curl http://localhost:8081/subjects/pg-users-value/versions/1 | python3 -m json.tool
curl http://localhost:8081/subjects/pg-users-value/versions/2 | python3 -m json.tool
# Version 2 should include the "age" field

# Check sink database - auto.evolve should have added the column
psql -U postgres -d connect_sink -c "SELECT * FROM \"pg-users\";"
# Frank should appear with age=30
```

---

## Step 7: Useful Operations

### Monitor Connectors

```bash
# List all connectors
curl http://localhost:8083/connectors

# Detailed status
curl http://localhost:8083/connectors/jdbc-source-users/status | python3 -m json.tool
curl http://localhost:8083/connectors/jdbc-sink-users/status | python3 -m json.tool
```

### Pause / Resume

```bash
# Pause source connector (stops reading from DB)
curl -X PUT http://localhost:8083/connectors/jdbc-source-users/pause

# Resume
curl -X PUT http://localhost:8083/connectors/jdbc-source-users/resume
```

### Update Connector Config

```bash
# Change poll interval to 10 seconds
curl -X PUT http://localhost:8083/connectors/jdbc-source-users/config \
    -H "Content-Type: application/json" \
    -d '{
        "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
        "connection.url": "jdbc:postgresql://localhost:5432/connect_source",
        "connection.user": "postgres",
        "connection.password": "postgres",
        "mode": "incrementing",
        "incrementing.column.name": "id",
        "topic.prefix": "pg-",
        "table.whitelist": "users",
        "poll.interval.ms": "10000"
    }'
```

### Delete Connectors

```bash
curl -X DELETE http://localhost:8083/connectors/jdbc-source-users
curl -X DELETE http://localhost:8083/connectors/jdbc-sink-users
```

### Cleanup

```bash
psql -U postgres -c "DROP DATABASE connect_source;"
psql -U postgres -c "DROP DATABASE connect_sink;"
```

---

## Troubleshooting

### Connector Stuck in "FAILED" State

```bash
# Check the error
curl http://localhost:8083/connectors/<name>/status | python3 -m json.tool
# Look at "trace" field for the stack trace
```

**Common issues:**
- Wrong JDBC URL or credentials
- PostgreSQL JDBC driver not found (check `plugin.path`)
- Table doesn't exist / column doesn't exist
- Schema Registry not reachable

```bash
# After fixing, restart:
curl -X POST http://localhost:8083/connectors/<name>/restart
```

### JDBC Driver Not Found

The PostgreSQL JDBC driver must be in the connector's plugin path. Check your connect worker config for `plugin.path` setting. Download `postgresql-xx.x.x.jar` and place it in:
```
$CONFLUENT_HOME/share/confluent-hub-components/confluentinc-kafka-connect-jdbc/lib/
```

### No Data in Sink

1. Check source connector is RUNNING
2. Check topic has data:
   ```bash
   kafka-avro-console-consumer --topic pg-users --from-beginning \
       --bootstrap-server localhost:9092 \
       --property schema.registry.url=http://localhost:8081
   ```
3. Check sink connector status for errors
4. Check sink DB table exists and has correct name (topic name = table name)

---

## Summary of Ports

| Service | Port |
|---------|------|
| Kafka Broker | `localhost:9092` |
| Schema Registry | `localhost:8081` |
| Kafka Connect | `localhost:8083` |
| PostgreSQL | `localhost:5432` |
