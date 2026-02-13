# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Learning and reference repository for Kafka Connect, focused exclusively on **JDBC connectors** (source and sink) with a local Confluent Platform and PostgreSQL setup. No Docker, no Debezium, no S3/Elasticsearch connectors, no custom connectors.

## Repository Structure

- `notes/` — Reference documentation as markdown:
  - `kafka_connect.md` — Core concepts: connectors, tasks, workers, standalone vs distributed mode, JDBC source/sink configuration, error handling, converters, Schema Registry, SMTs, connector internals
  - `hands_on_jdbc.md` — Step-by-step hands-on guide: setting up PostgreSQL source/sink DBs, starting Kafka + Schema Registry + Connect, creating JDBC source and sink connectors, verifying data flow, schema evolution
  - `rest_api.md` — Kafka Connect REST API (port 8083) and Schema Registry REST API (port 8081) endpoint reference
- `practice/` — Empty directory for hands-on exercises

## Local Environment

| Service          | Port   |
|------------------|--------|
| Kafka Broker     | 9092   |
| Schema Registry  | 8081   |
| Kafka Connect    | 8083   |
| PostgreSQL       | 5432   |

### Starting Services

```bash
# Kafka (KRaft mode)
kafka-server-start $CONFLUENT_HOME/etc/kafka/kraft/server.properties

# Schema Registry
schema-registry-start $CONFLUENT_HOME/etc/schema-registry/schema-registry.properties

# Kafka Connect (standalone with Avro)
connect-standalone $CONFLUENT_HOME/etc/schema-registry/connect-avro-standalone.properties
```

### Key Commands

```bash
# List connectors
curl http://localhost:8083/connectors

# Create a connector
curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" -d '{"name":"...","config":{...}}'

# Check connector status
curl http://localhost:8083/connectors/{name}/status | python3 -m json.tool

# Consume Avro messages
kafka-avro-console-consumer --bootstrap-server localhost:9092 --topic {topic} --from-beginning --property schema.registry.url=http://localhost:8081

# Schema Registry subjects
curl http://localhost:8081/subjects
```

## Constraints

- JDBC connectors only — no Debezium, S3, Elasticsearch, etc.
- No Docker — local Confluent Platform + PostgreSQL
- No custom connectors — only pre-built ones
- Keep edits focused and scoped to what was asked
