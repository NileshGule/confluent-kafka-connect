# JDBC Source for Kafka Connect

### create Docker network named `confluent`

```bash

docker network create confluent

```

## Start Zookeeper

```bash

docker run -d \
    --net=confluent \
    --name=zookeeper \
    -e ZOOKEEPER_CLIENT_PORT=32181 \
    -e ZOOKEEPER_TICK_TIME=2000 \
    confluentinc/cp-zookeeper:4.1.0

```

## Start Kafka

```bash

docker run -d \
    --net=confluent \
    --name=kafka \
    -e KAFKA_ZOOKEEPER_CONNECT=zookeeper:32181 \
    -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:29092 \
    -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
    confluentinc/cp-kafka:4.1.0

```

## Start Schema Registry

```bash

docker run -d \
  --net=confluent \
  --name=schema-registry \
  -e SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL=zookeeper:32181 \
  -e SCHEMA_REGISTRY_HOST_NAME=schema-registry \
  -e SCHEMA_REGISTRY_LISTENERS=http://0.0.0.0:8081 \
  confluentinc/cp-schema-registry:4.1.0

```

## Create Kafka topics

```bash

docker run \
  --net=confluent \
  --rm \
  confluentinc/cp-kafka:4.1.0 \
  kafka-topics --create --topic quickstart-avro-offsets --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:32181

  docker run \
  --net=confluent \
  --rm \
  confluentinc/cp-kafka:4.1.0 \
  kafka-topics --create --topic quickstart-avro-config --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:32181

  docker run \
  --net=confluent \
  --rm \
  confluentinc/cp-kafka:4.1.0 \
  kafka-topics --create --topic quickstart-avro-status --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:32181

  ```

## Copy MySQL JDBC driver to jars folder

```bash

mkdir -p /tmp/quicstart/jars

curl -k -SL "https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.39.tar.gz" | tar -xzf - -C /tmp/quickstart/jars --strip-components=1 mysql-connector-java-5.1.39/mysql-connector-java-5.1.39-bin.jar

```

## Start connect worker with Avro support

```bash

docker run -d \
  --name=kafka-connect-avro \
  --net=confluent \
  -e CONNECT_BOOTSTRAP_SERVERS=localhost:29092 \
  -e CONNECT_REST_PORT=28083 \
  -e CONNECT_GROUP_ID="quickstart-avro" \
  -e CONNECT_CONFIG_STORAGE_TOPIC="quickstart-avro-config" \
  -e CONNECT_OFFSET_STORAGE_TOPIC="quickstart-avro-offsets" \
  -e CONNECT_STATUS_STORAGE_TOPIC="quickstart-avro-status" \
  -e CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR=1 \
  -e CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR=1 \
  -e CONNECT_STATUS_STORAGE_REPLICATION_FACTOR=1 \
  -e CONNECT_KEY_CONVERTER="io.confluent.connect.avro.AvroConverter" \
  -e CONNECT_VALUE_CONVERTER="io.confluent.connect.avro.AvroConverter" \
  -e CONNECT_KEY_CONVERTER_SCHEMA_REGISTRY_URL="http://schema-registry:8081" \
  -e CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL="http://schema-registry:8081" \
  -e CONNECT_INTERNAL_KEY_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
  -e CONNECT_INTERNAL_VALUE_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
  -e CONNECT_REST_ADVERTISED_HOST_NAME="localhost" \
  -e CONNECT_LOG4J_ROOT_LOGLEVEL=DEBUG \
  -e CONNECT_LOG4J_LOGGERS=org.reflections=ERROR \
  -e CONNECT_PLUGIN_PATH=/usr/share/java,/etc/kafka-connect/jars \
  -v /tmp/quickstart/file:/tmp/quickstart \
  -v /tmp/quickstart/jars:/etc/kafka-connect/jars \
  confluentinc/cp-kafka-connect:latest

  docker run \
  --name=kafka-connect-avro \
  --net=confluent \
  -e CONNECT_BOOTSTRAP_SERVERS=kafka:29092 \
  -e CONNECT_REST_PORT=28083 \
  -e CONNECT_GROUP_ID="quickstart-avro" \
  -e CONNECT_CONFIG_STORAGE_TOPIC="quickstart-avro-config" \
  -e CONNECT_OFFSET_STORAGE_TOPIC="quickstart-avro-offsets" \
  -e CONNECT_STATUS_STORAGE_TOPIC="quickstart-avro-status" \
  -e CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR=1 \
  -e CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR=1 \
  -e CONNECT_STATUS_STORAGE_REPLICATION_FACTOR=1 \
  -e CONNECT_KEY_CONVERTER="io.confluent.connect.avro.AvroConverter" \
  -e CONNECT_VALUE_CONVERTER="io.confluent.connect.avro.AvroConverter" \
  -e CONNECT_KEY_CONVERTER_SCHEMA_REGISTRY_URL="http://schema-registry:8081" \
  -e CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL="http://schema-registry:8081" \
  -e CONNECT_INTERNAL_KEY_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
  -e CONNECT_INTERNAL_VALUE_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
  -e CONNECT_REST_ADVERTISED_HOST_NAME="kafka-connect" \
  -e CONNECT_LOG4J_ROOT_LOGLEVEL=DEBUG \
  -e CONNECT_LOG4J_LOGGERS=org.reflections=ERROR \
  -e CONNECT_PLUGIN_PATH=/usr/share/java,/etc/kafka-connect/jars \
  -v /tmp/quickstart/file:/tmp/quickstart \
  -v /tmp/quickstart/jars:/etc/kafka-connect/jars \
  confluentinc/cp-kafka-connect:4.1.0

```

## Start MySQL container

```bash

docker run -d \
  --name=quickstart-mysql \
  --net=confluent \
  -e MYSQL_ROOT_PASSWORD=confluent \
  -e MYSQL_USER=confluent \
  -e MYSQL_PASSWORD=confluent \
  -e MYSQL_DATABASE=connect_test \
  mysql

```

## Create Database

```bash

docker exec -it quickstart-mysql bash

mysql -u confluent -pconfluent

CREATE DATABASE IF NOT EXISTS connect_test;
USE connect_test;

DROP TABLE IF EXISTS test;


CREATE TABLE IF NOT EXISTS test (
  id serial NOT NULL PRIMARY KEY,
  name varchar(100),
  email varchar(200),
  department varchar(200),
  modified timestamp default CURRENT_TIMESTAMP NOT NULL,
  INDEX `modified_index` (`modified`)
);

INSERT INTO test (name, email, department) VALUES ('alice', 'alice@abc.com', 'engineering');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
exit;

```

## Create JDBC Source Connector

```bash

export CONNECT_HOST=localhost

docker exec kafka-connect-avro curl -X POST \
  -H "Content-Type: application/json" \
  --data '{ "name": "quickstart-jdbc-source", "config": { "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector", "tasks.max": 1, "connection.url": "jdbc:mysql://127.0.0.1:3306/connect_test?user=root&password=confluent", "mode": "incrementing", "incrementing.column.name": "id", "timestamp.column.name": "modified", "topic.prefix": "quickstart-jdbc-", "poll.interval.ms": 1000 } }' \
  http://localhost:28083/connectors

  docker exec kafka-connect-avro curl -X POST \
  -H "Content-Type: application/json" \
  --data '{ "name": "quickstart-jdbc-source", "config": { "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector", "tasks.max": 1, "connection.url": "jdbc:mysql://quickstart-mysql:3306/connect_test?user=confluent&password=confluent", "mode": "incrementing", "incrementing.column.name": "id", "timestamp.column.name": "modified", "topic.prefix": "quickstart-jdbc-", "poll.interval.ms": 1000 } }' \
  http://localhost:28083/connectors

  curl -X POST \
  -H "Content-Type: application/json" \
  --data '{ "name": "quickstart-jdbc-source", "config": { "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector", "tasks.max": 1, "connection.url": "jdbc:mysql://quickstart-mysql:3306/connect_test?user=confluent&password=confluent", "mode": "incrementing", "incrementing.column.name": "id", "timestamp.column.name": "modified", "topic.prefix": "quickstart-jdbc-", "poll.interval.ms": 1000 } }' \
  http://localhost:28083/connectors

  docker exec -it kafka-connect-avro bash

```

