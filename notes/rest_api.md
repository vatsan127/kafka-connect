# Kafka Connect REST API

Kafka Connect exposes a REST API for management on port **8083**.

---

## Cluster Information

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/` | Connect version info |
| `GET` | `/connectors` | List all connectors |
| `GET` | `/connector-plugins` | List available connector plugins |

```bash
curl http://localhost:8083/
curl http://localhost:8083/connectors
curl http://localhost:8083/connector-plugins | python3 -m json.tool
```

---

## Connector Management

### Create Connector

```bash
curl -X POST http://localhost:8083/connectors \
    -H "Content-Type: application/json" \
    -d '{
        "name": "my-source-connector",
        "config": {
            "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
            "connection.url": "jdbc:postgresql://localhost:5432/mydb",
            "connection.user": "user",
            "connection.password": "password",
            "mode": "incrementing",
            "incrementing.column.name": "id",
            "topic.prefix": "pg-",
            "table.whitelist": "users"
        }
    }'
```

### Get Connector

```bash
curl http://localhost:8083/connectors/{name}
```

### Get Connector Config

```bash
curl http://localhost:8083/connectors/{name}/config
```

### Update Connector

```bash
curl -X PUT http://localhost:8083/connectors/{name}/config \
    -H "Content-Type: application/json" \
    -d '{
        "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
        "connection.url": "jdbc:postgresql://localhost:5432/mydb",
        "connection.user": "user",
        "connection.password": "password",
        "mode": "incrementing",
        "incrementing.column.name": "id",
        "topic.prefix": "pg-",
        "table.whitelist": "users",
        "poll.interval.ms": "10000"
    }'
```

### Delete Connector

```bash
curl -X DELETE http://localhost:8083/connectors/{name}
```

### Get Connector Status

```bash
curl http://localhost:8083/connectors/{name}/status | python3 -m json.tool
```

---

## Task Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/connectors/{name}/tasks` | List tasks |
| `GET` | `/connectors/{name}/tasks/{taskId}/status` | Get task status |
| `POST` | `/connectors/{name}/tasks/{taskId}/restart` | Restart task |

```bash
curl http://localhost:8083/connectors/{name}/tasks | python3 -m json.tool
curl http://localhost:8083/connectors/{name}/tasks/0/status | python3 -m json.tool
curl -X POST http://localhost:8083/connectors/{name}/tasks/0/restart
```

---

## Connector Control

| Method | Endpoint | Description |
|--------|----------|-------------|
| `PUT` | `/connectors/{name}/pause` | Pause connector |
| `PUT` | `/connectors/{name}/resume` | Resume connector |
| `POST` | `/connectors/{name}/restart` | Restart connector |

```bash
# Pause
curl -X PUT http://localhost:8083/connectors/{name}/pause

# Resume
curl -X PUT http://localhost:8083/connectors/{name}/resume

# Restart
curl -X POST http://localhost:8083/connectors/{name}/restart
```

---

## Schema Registry REST API

Schema Registry runs on port **8081**.

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/subjects` | List all subjects |
| `GET` | `/subjects/{subject}/versions` | List schema versions |
| `GET` | `/subjects/{subject}/versions/latest` | Get latest schema |
| `GET` | `/subjects/{subject}/versions/{version}` | Get specific version |
| `GET` | `/schemas/ids/{id}` | Get schema by global ID |
| `GET` | `/config` | Get global compatibility level |
| `PUT` | `/config` | Set global compatibility level |

```bash
# List all subjects
curl http://localhost:8081/subjects

# Get latest schema for a topic's value
curl http://localhost:8081/subjects/pg-users-value/versions/latest | python3 -m json.tool

# List all versions
curl http://localhost:8081/subjects/pg-users-value/versions

# Get schema by global ID
curl http://localhost:8081/schemas/ids/1 | python3 -m json.tool

# Get compatibility level
curl http://localhost:8081/config

# Set compatibility level
curl -X PUT http://localhost:8081/config \
    -H "Content-Type: application/json" \
    -d '{"compatibility": "BACKWARD"}'
```
