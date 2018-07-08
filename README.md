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
