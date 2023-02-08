# Debezium Kafka Connect SQL Server

## Table of Contents

- [About](#about)
- [Getting Started](#getting_started)
    - [Configure and install Debezium Kafka](#configure-install-debezium-kafka)
- [Usage](#usage)
- [Contributing](../CONTRIBUTING.md)

## About <a name = "about"></a>

Docker CDC SQL Server database using Debezium Kafka 

## Getting Started <a name = "getting_started"></a>

These instructions will get you a copy of the project up and running on your local machine for development and testing purposes. See [deployment](#deployment) for notes on how to deploy the project on a live system.

### Prerequisites

Installed
- Docker
- Docker Compose
- SQL Server
- SQL Server Management Studio (SSMS)
- Postman

### Configure and install Debezium Kafka <a name = "configure-install-debezium-kafka"></a>

1. Deploy stack

Create file docker-compose.yaml

```yml
version: "3"

services:
  zookeeper:
    image: debezium/zookeeper
    ports:
      - "2181:2181"
      - "2889:2888"
      - "3889:3888"

  # Broker
  kafka:
    image: debezium/kafka
    ports:
      - "9092:9092"
      - "29092:29092"
    depends_on:
      - zookeeper
    environment:
      - ZOOKEEPER_CONNECT=zookeeper:2181
      - KAFKA_ADVERTISED_LISTENERS=LISTENER_EXT://localhost:29092,LISTENER_INT://kafka:9092
      - KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=LISTENER_INT:PLAINTEXT,LISTENER_EXT:PLAINTEXT
      - KAFKA_LISTENERS=LISTENER_INT://0.0.0.0:9092,LISTENER_EXT://0.0.0.0:29092
      - KAFKA_INTER_BROKER_LISTENER_NAME=LISTENER_INT

  connect:
    image: debezium/connect
    ports:
      - "8083:8083"
    environment:
      - BOOTSTRAP_SERVERS=kafka:9092
      - GROUP_ID=1
      - CONFIG_STORAGE_TOPIC=my_connect_configs
      - OFFSET_STORAGE_TOPIC=my_connect_offsets
      - STATUS_STORAGE_TOPIC=my_connect_statuses
    depends_on:
      - zookeeper
      - kafka
    volumes:
      - volume_kafka_connect:/kafka

  kafka-connect-ui:
    image: landoop/kafka-connect-ui
    hostname: kafka-connect-ui
    ports:
      - "8100:8000"
    depends_on:
      - connect
    environment:
      CONNECT_URL: "connect:8083"
      PROXY: "true"

  schema-registry:
    image: confluentinc/cp-schema-registry
    ports:
      - "8085:8085"
    hostname: schema-registry
    depends_on:
      - zookeeper
      - kafka
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      # SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL: "zookeeper:2181"
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka:9092
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8085

  rest-proxy:
    image: confluentinc/cp-kafka-rest
    ports:
      - "8082:8082"
    hostname: rest-proxy
    depends_on:
      - zookeeper
      - kafka
      - schema-registry
    environment:
      KAFKA_REST_HOST_NAME: rest-proxy
      KAFKA_REST_LISTENERS: http://0.0.0.0:8082
#      KAFKA_REST_SCHEMA_REGISTRY_URL: 192.168.100.14:8081
#      KAFKA_REST_ZOOKEEPER_CONNECT: 192.168.100.14:2181
      KAFKA_REST_BOOTSTRAP_SERVERS: http://10.10.65.1:9092

  kafka-topics-ui:
    image: landoop/kafka-topics-ui
    ports:
      - "8200:8000"
    hostname: kafka-topics-ui
    depends_on:
      - rest-proxy
    environment:
      KAFKA_REST_PROXY_URL: http://rest-proxy:8082
      PROXY: "true"

  kafka_manager:
    image: hlebalbau/kafka-manager
    ports:
      - "9000:9000"
    depends_on:
      - zookeeper
    environment:
      ZK_HOSTS: zookeeper:2181
      APPLICATION_SECRET: random-secret

volumes:
  volume_kafka_connect:
    external: true

```

Save and run.

```bash
sudo docker stack deploy -c docker-compose.yml <stack-name>
```
2. Configure java.security Docker container

Go to docker volume kafka connect directory and find java.security file

```bash
find <volume-dir> -name java.security
```
Edit file, remove 3DES_EDE_CBC in jdk.tls.disabledAlgorithms key.

Save and restart container
```bash
docker restart <container-id>
```
3. Configure cdc SQL Server

Open SSMS, Login and create query workspace

```sql
USE <database>;
EXEC sys.sp_cdc_enable_db
EXEC sys.sp_cdc_help_change_data_capture

EXEC sys.sp_cdc_enable_table
	@source_schema = N'dbo',
	@source_name = N'users',
	@role_name = NULL

select
  name,
  is_cdc_enabled
from sys.databases WHERE name = '<database>';

SELECT
	name,
	is_tracked_by_cdc
	from sys.tables;
```


## Usage <a name = "usage"></a>

Test using Postman
https://documenter.getpostman.com/view/10123708/2s935oLPhz

1. Create connector

- url : {{HOST:8083}}/connectors

- body :
```json
{
    "name": "sqlserver-connector",
    "config": {
        "connector.class": "io.debezium.connector.sqlserver.SqlServerConnector",
        "tasks.max": "1",
        "topic.prefix": "sqlserver-test",
        "database.hostname": "<sqlserver-host>",
        "database.port": "1433",
        "database.user": "<db-user>",
        "database.password": "<db-password>",
        "database.names" : "<db-name>",
        "database.server.name": "fullfillment_sqlserver-test",
        "database.whitelist": "<db-names>",
        "table.include.list": "cdc.users",
        "database.history.kafka.bootstrap.servers": "10.10.65.1:9092",
        "database.history.kafka.topic": "sqlserver-sj.fullfillment",
        "database.encrypt": false
    }
}
```
