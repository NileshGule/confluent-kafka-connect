# Repository to try out Confluent Kafka Connect features

A quick test of Confluent Kafka connect features using single node kafka cluster running in Docker

## Docker setup

Follow the steps mentioned in the docs to get started with Docker version of Confluent Kafka Connect

[Getting Started FileSource Example](https://docs.confluent.io/current/installation/docker/docs/quickstart.html#getting-started-with-docker-client)

## Commands

### Run Docker compose up after cloning the confluent repo

```bash

docker-compose up -d

```

### Verify contaners are up

```bash

docker-compose ps

```

### Verify Zookeeper is healthy

```bash

docker-compose logs zookeeper | grep -i binding

```

### verify kafka broker is healthy

```bash

docker-compose logs kafka | grep -i started

```

### Stop compose

```bash

docker-compose stop

```

## Run simple File based connector

### create Docker network named `confluent`

```bash

docker network create confluent

```

### Start Zookeeper

```bash

docker run \
    --net=confluent \
    --name=zookeeper \
    -e ZOOKEEPER_CLIENT_PORT=2181 \
    confluentinc/cp-zookeeper:4.1.0

```

### verify logs

```bash

docker logs zokeeper

```

### Start Kafka

```bash

docker run -d \
    --net=confluent \
    --name=kafka \
    -e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
    -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092 \
    -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
    confluentinc/cp-kafka:4.1.0

```

### verify kafka logs

```bash

docker logs kafka

```

### Produce dummy `foo` topic on Kafka

```bash

docker run \
  --net=confluent \
  --rm confluentinc/cp-kafka:4.1.0 \
  kafka-topics --create --topic foo --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181

```

### Verify the topic was created

```bash

docker run \
  --net=confluent \
  --rm \
  confluentinc/cp-kafka:4.1.0 \
  kafka-topics --describe --topic foo --zookeeper zookeeper:2181

```

### Publish data to `foo` topic

```bash

docker run \
  --net=confluent \
  --rm \
  confluentinc/cp-kafka:4.1.0 \
  bash -c "seq 42 | kafka-console-producer --request-required-acks 1 --broker-list kafka:9092 --topic foo && echo 'Produced 42 messages.'"

```

### Read back the messages from `foo` topic

```bash

docker run \
  --net=confluent \
  --rm \
  confluentinc/cp-kafka:4.1.0 \
  kafka-console-consumer --bootstrap-server kafka:9092 --topic foo --from-beginning --max-messages 42

```

### Start Schema Registry container

```bash

docker run -d \
  --net=confluent \
  --name=schema-registry \
  -e SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL=zookeeper:2181 \
  -e SCHEMA_REGISTRY_HOST_NAME=schema-registry \
  -e SCHEMA_REGISTRY_LISTENERS=http://0.0.0.0:8081 \
  confluentinc/cp-schema-registry:4.1.0

```

### Verify Schema Registry logs

```bash

docker logs schema-registry

```

### Publish Data using Schema Registry

```bash

docker run -it --net=confluent --rm confluentinc/cp-schema-registry:4.1.0 bash

/usr/bin/kafka-avro-console-producer \
  --broker-list kafka:9092 --topic bar \
  --property schema.registry.url=http://schema-registry:8081 \
  --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"f1","type":"string"}]}'

```

### REST Proxy

```bash

docker run -d \
  --net=confluent \
  --name=kafka-rest \
  -e KAFKA_REST_ZOOKEEPER_CONNECT=zookeeper:2181 \
  -e KAFKA_REST_LISTENERS=http://0.0.0.0:8082 \
  -e KAFKA_REST_SCHEMA_REGISTRY_URL=http://schema-registry:8081 \
  -e KAFKA_REST_HOST_NAME=kafka-rest \
  confluentinc/cp-kafka-rest:4.1.0

  docker run -it --net=confluent --rm confluentinc/cp-schema-registry:4.1.0 bash

  curl -X POST -H "Content-Type: application/vnd.kafka.v1+json" \
  --data '{"name": "my_consumer_instance", "format": "avro", "auto.offset.reset": "smallest"}' \
  http://kafka-rest:8082/consumers/my_avro_consumer

  curl -X GET -H "Accept: application/vnd.kafka.avro.v1+json" \
  http://kafka-rest:8082/consumers/my_avro_consumer/instances/my_consumer_instance/topics/bar

```

## Kafka Connect

### Create `quickstart-offsets` topic

```bash

docker run \
  --net=confluent \
  --rm \
  confluentinc/cp-kafka:4.1.0 \
  kafka-topics --create --topic quickstart-offsets --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181

```

### Create `quickstart-data` topic

```bash

docker run \
  --net=confluent \
  --rm \
  confluentinc/cp-kafka:4.1.0 \
  kafka-topics --create --topic quickstart-data --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181

```

### Create `quickstart-status` topic

```bash

docker run \
  --net=confluent \
  --rm \
  confluentinc/cp-kafka:4.1.0 \
  kafka-topics --create --topic quickstart-status --partitions 1 --replication-factor 1 --if-not-exists --zookeeper zookeeper:2181

```

### Start Kafka connect worker using the 3 topics

```bash

docker run -d \
  --name=kafka-connect \
  --net=confluent \
  -e CONNECT_PRODUCER_INTERCEPTOR_CLASSES=io.confluent.monitoring.clients.interceptor.MonitoringProducerInterceptor \
  -e CONNECT_CONSUMER_INTERCEPTOR_CLASSES=io.confluent.monitoring.clients.interceptor.MonitoringConsumerInterceptor \
  -e CONNECT_BOOTSTRAP_SERVERS=kafka:9092 \
  -e CONNECT_REST_PORT=8082 \
  -e CONNECT_GROUP_ID="quickstart" \
  -e CONNECT_CONFIG_STORAGE_TOPIC="quickstart-config" \
  -e CONNECT_OFFSET_STORAGE_TOPIC="quickstart-offsets" \
  -e CONNECT_STATUS_STORAGE_TOPIC="quickstart-status" \
  -e CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR=1 \
  -e CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR=1 \
  -e CONNECT_STATUS_STORAGE_REPLICATION_FACTOR=1 \
  -e CONNECT_KEY_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
  -e CONNECT_VALUE_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
  -e CONNECT_INTERNAL_KEY_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
  -e CONNECT_INTERNAL_VALUE_CONVERTER="org.apache.kafka.connect.json.JsonConverter" \
  -e CONNECT_REST_ADVERTISED_HOST_NAME="kafka-connect" \
  -e CONNECT_LOG4J_ROOT_LOGLEVEL=DEBUG \
  -e CONNECT_LOG4J_LOGGERS=org.reflections=ERROR \
  -e CONNECT_PLUGIN_PATH=/usr/share/java \
  -e CONNECT_REST_HOST_NAME="kafka-connect" \
  -v /tmp/quickstart/file:/tmp/quickstart \
  confluentinc/cp-kafka-connect:4.1.0

```

### Verify Worker logs

```bash

docker logs kafka-connect | grep started

```

### Create directory to store input data

```bash

docker exec kafka-connect mkdir -p /tmp/quickstart/file

```

### Add dummy data to file `input.txt`

```bash

docker exec kafka-connect sh -c 'seq 1000 > /tmp/quickstart/file/input.txt'

```

### Create FileSource connector

```bash

docker exec kafka-connect curl -s -X POST \
  -H "Content-Type: application/json" \
  --data '{"name": "quickstart-file-source", "config": {"connector.class":"org.apache.kafka.connect.file.FileStreamSourceConnector", "tasks.max":"1", "topic":"quickstart-data", "file": "/tmp/quickstart/file/input.txt"}}' \
  http://kafka-connect:8082/connectors

```

### Check the status of connector

```bash

docker exec kafka-connect curl -s -X GET http://kafka-connect:8082/connectors/quickstart-file-source/status

```

### Create FileSink

```bash

docker exec kafka-connect curl -X POST -H "Content-Type: application/json" \
    --data '{"name": "quickstart-file-sink", "config": {"connector.class":"org.apache.kafka.connect.file.FileStreamSinkConnector", "tasks.max":"1", "topics":"quickstart-data", "file": "/tmp/quickstart/file/output.txt"}}' \
    http://kafka-connect:8082/connectors

```

## Cleanup

### Delete all the running containers

```bash

docker rm -f $(docker ps -a -q)

yes | docker system prune

```