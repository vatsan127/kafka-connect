# Kafka Connect

## 1. What is Kafka Connect?

### Definition
- Framework for streaming data between Kafka and external systems
- Scalable and fault-tolerant data integration
- No custom code required for common integrations

### Purpose
- Import data INTO Kafka (Source Connectors)
- Export data FROM Kafka (Sink Connectors)
- Standardized way to move data

### Architecture

```
┌──────────────┐                              ┌──────────────┐
│   External   │                              │   External   │
│   Systems    │                              │   Systems    │
│ (DB, Files)  │                              │ (DB, S3, ES) │
└──────┬───────┘                              └──────▲───────┘
       │                                             │
       │ Source                              Sink    │
       │ Connector                        Connector  │
       ▼                                             │
┌────────────────────────────────────────────────────────────┐
│                       KAFKA CONNECT                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   Worker    │  │   Worker    │  │   Worker    │        │
│  │  (Tasks)    │  │  (Tasks)    │  │  (Tasks)    │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────┐
│                       KAFKA CLUSTER                        │
│       Topics: data, connect-offsets, connect-configs       │
└────────────────────────────────────────────────────────────┘
```

### Why Kafka Connect?

**Without Connect:**
- Write custom producer/consumer code
- Handle failures, restarts, scaling manually
- Reinvent offset tracking

**With Connect:**
- Pre-built connectors for common systems
- Automatic offset management
- Distributed, scalable, fault-tolerant
- Declarative configuration

---

## 2. Core Concepts

### Connectors

**Source Connector:**
- Reads from external system
- Writes to Kafka topics
- Examples: JDBC, Debezium (CDC), File

**Sink Connector:**
- Reads from Kafka topics
- Writes to external system
- Examples: JDBC, Elasticsearch, S3, HDFS

### Tasks
- Unit of parallelism
- Connector divides work into tasks
- Each task handles subset of data
- Example: JDBC connector creates task per table

### Workers

Workers are JVM processes that execute connectors and tasks. They are the runtime engine of Kafka Connect. A worker does the following:
- Receives connector configurations
- Instantiates connector and task classes
- Manages the lifecycle (start, stop, restart)
- Handles converter serialization/deserialization
- Commits offsets after successful processing
- Exposes the REST API on port 8083

There are two deployment modes:

**Standalone Mode:**
- Single worker process running on one machine
- Offsets stored in a **local file** on disk
- Connectors are passed as `.properties` files at startup
- If the process dies, everything stops — no failover
- Good for: development, testing, simple single-machine pipelines

**Distributed Mode:**
- Multiple worker processes form a **Connect cluster**
- Workers discover each other via the same `group.id`
- Offsets, configs, and status stored in **internal Kafka topics** (durable + replicated)
- Connectors are submitted via REST API (not properties files)
- If a worker dies, its connectors/tasks are **rebalanced** to surviving workers automatically
- Good for: production, high availability, scaling

**How Rebalancing Works (Distributed):**
1. Worker joins or leaves the group
2. Connect triggers a rebalance (like consumer group rebalance)
3. Connectors and tasks are redistributed evenly across remaining workers
4. Tasks resume from their last committed offset — no data loss

**What Lives Inside a Worker:**

```
┌────────────────────────────────────────────────────┐
│                       WORKER                       │
│                                                    │
│  ┌──────────────────────────────────────────────┐  │
│  │  Connector Instance                          │  │
│  │  - Validates config                          │  │
│  │  - Generates task configs                    │  │
│  │  - Monitors task health                      │  │
│  └──────────────────────────────────────────────┘  │
│                                                    │
│  ┌──────────────┐  ┌──────────────┐                │
│  │   Task 0     │  │   Task 1     │  ...           │
│  │  (thread)    │  │  (thread)    │                │
│  └──────────────┘  └──────────────┘                │
│                                                    │
│  ┌──────────────────────────────────────────────┐  │
│  │  Converter (key + value)                     │  │
│  │  REST API Handler (:8083)                    │  │
│  │  Offset Manager                              │  │
│  └──────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────┘
```

- **One connector instance** per connector config — it doesn't move data itself
- **Multiple tasks** (threads) do the actual data movement
- A single worker can run many connectors, each with many tasks

---

## 3. Standalone Deployment

**Characteristics:**
- Single worker process
- Offsets stored in local file
- Connectors configured via `.properties` files at startup

**Use Cases:**
- Development and testing
- Simple pipelines
- Edge deployments

**Start Command:**
```bash
connect-standalone.sh \
    config/connect-standalone.properties \
    config/source-connector.properties
```

**Configuration** (`connect-standalone.properties`):
```properties
bootstrap.servers=localhost:9092
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
offset.storage.file.filename=/tmp/connect.offsets
```

**Standalone vs Distributed:**

| Feature | Standalone | Distributed |
| --- | --- | --- |
| Workers | Single process | Multiple processes (cluster) |
| Offset storage | Local file | Kafka topic (`connect-offsets`) |
| Config storage | `.properties` files | Kafka topic (`connect-configs`) |
| Fault tolerance | None — process dies, work stops | Auto-rebalance to other workers |
| Submit connectors | CLI args at startup | REST API (runtime) |
| Use case | Dev / testing | Production |

---

## 4. JDBC Connectors

### JDBC Source Connector

**Purpose:**
- Reads from relational databases
- Supports bulk and incremental modes
- Tables become topics

**Modes:**

| Mode | Description |
| --- | --- |
| `bulk` | Load entire table each time |
| `incrementing` | Track new rows by incrementing column (id) |
| `timestamp` | Track new/updated rows by timestamp column |
| `timestamp+incrementing` | Combination of both |

**Configuration:**
```properties
connector.class=io.confluent.connect.jdbc.JdbcSourceConnector
connection.url=jdbc:postgresql://localhost:5432/mydb
connection.user=user
connection.password=password
mode=incrementing
incrementing.column.name=id
topic.prefix=postgres-
table.whitelist=users,orders
poll.interval.ms=5000
```

**Key Parameters:**

| Parameter | Description |
| --- | --- |
| `mode` | `bulk`, `incrementing`, `timestamp`, `timestamp+incrementing` |
| `incrementing.column.name` | Column to track new rows |
| `timestamp.column.name` | Column to track updates |
| `topic.prefix` | Prefix for created topics |
| `table.whitelist` | Tables to include |
| `poll.interval.ms` | How often to poll database |

### JDBC Sink Connector

**Purpose:**
- Writes to relational databases
- Auto-creates tables (optional)
- Supports insert and upsert modes

**Insert Modes:**

| Mode | Description |
| --- | --- |
| `insert` | Simple INSERT statements |
| `upsert` | INSERT with ON CONFLICT UPDATE (or equivalent) |
| `update` | Only UPDATE existing rows |

**Configuration:**
```properties
connector.class=io.confluent.connect.jdbc.JdbcSinkConnector
connection.url=jdbc:postgresql://localhost:5432/mydb
connection.user=user
connection.password=password
topics=orders
auto.create=true
auto.evolve=true
insert.mode=upsert
pk.mode=record_key
pk.fields=id
batch.size=3000
```

**Key Parameters:**

| Parameter | Description |
| --- | --- |
| `auto.create` | Auto-create tables if not exists |
| `auto.evolve` | Auto-add columns for new fields |
| `insert.mode` | `insert`, `upsert`, `update` |
| `pk.mode` | `record_key`, `record_value`, `kafka` |
| `pk.fields` | Primary key field names |
| `batch.size` | Number of records per batch |

---

## 5. Error Handling

### Error Types

**Connector Errors:**
- Configuration errors
- Connection failures
- Authentication issues

**Task Errors:**
- Data format errors
- Serialization failures
- External system errors

**Transient vs Permanent:**
- Transient: Network timeout, temporary unavailability
- Permanent: Invalid data, schema mismatch

### Error Handling Configuration

| Property | Values | Description |
| --- | --- | --- |
| `errors.tolerance` | `none` (default), `all` | `none` = fail on first error, `all` = skip bad records |
| `errors.log.enable` | `true`, `false` | Log errors |
| `errors.log.include.messages` | `true`, `false` | Include failed message in log |
| `errors.deadletterqueue.topic.name` | topic name | Topic for failed records |

### Example - Tolerant Connector

```json
{
    "name": "tolerant-sink",
    "config": {
        "connector.class": "JdbcSinkConnector",
        "errors.tolerance": "all",
        "errors.log.enable": "true",
        "errors.log.include.messages": "true",
        "errors.deadletterqueue.topic.name": "dlq-jdbc-sink",
        "errors.deadletterqueue.topic.replication.factor": "3"
    }
}
```

### Dead Letter Queue

**Purpose:**
- Capture failed records
- Allow inspection and debugging
- Enable reprocessing after fix

**DLQ Record Contains:**
- Original record
- Error details
- Timestamp
- Connector/task information

---

## 6. Transforms (Single Message Transforms)

Modify records one at a time as they flow through Connect.

```
Source: DB -> Connector -> [Transforms] -> Converter -> Kafka
Sink:  Kafka -> Converter -> [Transforms] -> Connector -> DB
```

### Configuration Pattern

```json
{
    "transforms": "alias1,alias2",
    "transforms.alias1.type": "<transform class>",
    "transforms.alias1.<config>": "<value>"
}
```

### Useful Transforms for JDBC

**ValueToKey** - Set message key from a value field:
```json
{
    "transforms": "createKey",
    "transforms.createKey.type": "org.apache.kafka.connect.transforms.ValueToKey",
    "transforms.createKey.fields": "id"
}
```

**ExtractField** - Pull single field out of struct (useful for key):
```json
{
    "transforms": "extractId",
    "transforms.extractId.type": "org.apache.kafka.connect.transforms.ExtractField$Key",
    "transforms.extractId.field": "id"
}
```

**ReplaceField** - Include/exclude/rename fields:
```json
{
    "transforms": "dropFields",
    "transforms.dropFields.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
    "transforms.dropFields.exclude": "internal_column,temp_field"
}
```

**MaskField** - Mask sensitive data:
```json
{
    "transforms": "mask",
    "transforms.mask.type": "org.apache.kafka.connect.transforms.MaskField$Value",
    "transforms.mask.fields": "ssn,credit_card"
}
```

> **Note:** Most transforms have `$Key` and `$Value` variants.

---

## 7. Connector Internals

### Lifecycle
1. Submit connector config (REST API or properties file)
2. Connect validates config
3. `Connector.start()` called
4. Connector splits work into tasks (e.g., one task per table)
5. Each `Task.start(config)` initializes
   - **Source:** `poll()` called repeatedly -> returns records
   - **Sink:** `put(records)` called with batches -> writes to DB
6. On shutdown: `Task.stop()` then `Connector.stop()`

### Offset Management

**Source Connector:**
- Tracks "where am I reading from" per table
- Stored AFTER successful Kafka produce
- Standalone: local file (`offset.storage.file.filename`)
- On restart: reads last offset, resumes from there
- Example: JDBC source stores last seen ID or timestamp

**Sink Connector:**
- Uses Kafka consumer group offsets
- Consumer group = `connect-{connector-name}`
- Commits offset after `put()` succeeds

