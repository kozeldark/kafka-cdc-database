# kafka-cdc-database
**CDC Sink Database 아키텍쳐**

![Untitled](https://user-images.githubusercontent.com/77625823/205200146-ce75ef3a-d728-41de-a570-78aee99c4963.png)


**환경**

KiC - CentOS 7.9

![Untitled (1)](https://user-images.githubusercontent.com/77625823/205200338-3fcebe94-0230-4565-be93-979dd8f76431.png)


docker compose

```jsx
version: '2'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    container_name: kafka
    image: confluentinc/cp-enterprise-kafka:latest
    depends_on:
      - zookeeper
    ports:
    # Exposes 9092 for external connections to the broker
    # Use kafka:29092 for connections internal on the docker network
      - 9092:9092
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      KAFKA_METRIC_REPORTERS: io.confluent.metrics.reporter.ConfluentMetricsReporter
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 100
      CONFLUENT_METRICS_REPORTER_BOOTSTRAP_SERVERS: kafka:29092
      CONFLUENT_METRICS_REPORTER_ZOOKEEPER_CONNECT: zookeeper:2181
      CONFLUENT_METRICS_REPORTER_TOPIC_REPLICAS: 1
      CONFLUENT_METRICS_ENABLE: 'true'
      CONFLUENT_SUPPORT_CUSTOMER_ID: 'anonymous'

  schema-registry:
    container_name: schema-registry
    image: confluentinc/cp-schema-registry:latest
    ports:
      - 8081:8081
    depends_on:
      - zookeeper
      - kafka
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL: zookeeper:2181

  kafka-connect-01:
    container_name: kafka-connect
    image: confluentinc/cp-kafka-connect:latest
    depends_on:
      - zookeeper
      - kafka
      - schema-registry
      - mysql
      - postgres
      - mongodb
    ports:
      - 8083:8083
    environment:
      CONNECT_LOG4J_APPENDER_STDOUT_LAYOUT_CONVERSIONPATTERN: "[%d] %p %X{connector.context}%m (%c:%L)%n"
      CONNECT_BOOTSTRAP_SERVERS: "kafka:29092"
      CONNECT_REST_PORT: 8083
      CONNECT_REST_ADVERTISED_HOST_NAME: "kafka-connect-01"
      CONNECT_GROUP_ID: compose-connect-group
      CONNECT_CONFIG_STORAGE_TOPIC: docker-connect-configs
      CONNECT_OFFSET_STORAGE_TOPIC: docker-connect-offsets
      CONNECT_STATUS_STORAGE_TOPIC: docker-connect-status
      CONNECT_KEY_CONVERTER: io.confluent.connect.avro.AvroConverter
      CONNECT_KEY_CONVERTER_SCHEMA_REGISTRY_URL: 'http://schema-registry:8081'
      CONNECT_VALUE_CONVERTER: io.confluent.connect.avro.AvroConverter
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: 'http://schema-registry:8081'
      CONNECT_INTERNAL_KEY_CONVERTER: "org.apache.kafka.connect.json.JsonConverter"
      CONNECT_INTERNAL_VALUE_CONVERTER: "org.apache.kafka.connect.json.JsonConverter"
      CONNECT_LOG4J_ROOT_LOGLEVEL: "INFO"
      CONNECT_LOG4J_LOGGERS: "org.apache.kafka.connect.runtime.rest=WARN,org.reflections=ERROR"
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: "1"
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: "1"
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: "1"
      CONNECT_PLUGIN_PATH: '/usr/share/java,/usr/share/confluent-hub-components,/usr/local/share/kafka/plugins'
    command: 
      - /bin/bash
      - -c 
      - |
				confluent-hub install --no-prompt confluentinc/kafka-connect-jdbc:latest
        confluent-hub install --no-prompt mongodb/kafka-connect-mongodb:latest
        # MySQL
        cd /usr/share/confluent-hub-components/confluentinc-kafka-connect-jdbc/lib
        curl https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.0.31/mysql-connector-j-8.0.31.jar --output mysql-connector-java-8.0.31.jar 

        # Now launch Kafka Connect
        sleep infinity &
        /etc/confluent/docker/run 

  kafdrop:
    container_name: kafdrop
    image: obsidiandynamics/kafdrop
    restart: "no"
    ports:
      - "9000:9000"
    environment:
      KAFKA_BROKERCONNECT: "kafka:29092"
      JVM_OPTS: "-Xms16M -Xmx48M -Xss180K -XX:-TieredCompilation -XX:+UseStringDeduplication -noverify"
      SCHEMAREGISTRY_CONNECT: "http://schema-registry:8081"
    depends_on:
      - "kafka"

# databases
  postgres:
    image: postgres:latest
    container_name: kafka-postgres
    restart: always
    privileged: true
    ports:
      - 5432:5432
    environment:
      POSTGRES_USER: admin 
      POSTGRES_PASSWORD: 12345
      POSTGRES_DB: target
 
  mongodb:
    container_name: kafka-mongo
    image: mongo:latest
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: 12345
      MONGO_INITDB_DATABASE: kafka-test
    ports:
      - 27017:27017
    volumes:
      - mongodb_data_container:/data/mongodb

  mysql:
    image: mysql:8.0
    container_name: kafka-mysql
    privileged: true
    ports:
      - 3306:3306
    environment:
      MYSQL_ROOT_PASSWORD: 12345

    command: 
      - bash 
      - -c 
      - |
        cat>/docker-entrypoint-initdb.d/z99_dump.sql <<EOF
        CREATE DATABASE source;
        USE source;

        CREATE TABLE source.user_table
        (
            id            int(11) PRIMARY KEY AUTO_INCREMENT NOT NULL,
            name  varchar(100) NOT NULL,
            date_field    date NOT NULL,
            email  varchar(100) NOT NULL,
            created_at    datetime DEFAULT CURRENT_TIMESTAMP,
            updated_at    datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP NOT NULL
        );

        INSERT INTO source.user_table(name, date_field, email)
        VALUES ('ddat', '2022-12-01', 'ddat@ddat.com'),
               ('chan', '2022-12-02', 'fasvvc@gmail.com'),
               ('woo', '2022-12-03', 'woo@woo.com'),
               ('kim', '2022-12-04', 'kim@kim.com'),
               ('lee', '2022-12-05', 'lee@lee.com');

        ALTER TABLE source.user_table 
        MODIFY COLUMN updated_at datetime 
        DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP NOT NULL;

        EOF
        # Launch mysql
        docker-entrypoint.sh mysqld
        #
        sleep infinity

volumes:
  mongodb_data_container:
```

모두 설치하면 다음과 같음

![Untitled (2)](https://user-images.githubusercontent.com/77625823/205200417-89de41b0-1163-424e-aafe-42b1e420c02f.png)


jq 설치

```jsx
sudo yum install -y epel-release
sudo yum install -y jq
```

jq를 이용하여 현재 플러그인이 잘 됐는지 확인

```jsx
curl localhost:8083/connector-plugins | jq
```

![Untitled (3)](https://user-images.githubusercontent.com/77625823/205200440-57d8139e-2fa0-4e19-ae80-85201e335ce0.png)


MYSQL 소스 DB 설정

```jsx
curl -X POST \
  -H "Content-Type: application/json" \
  --data '{ "name": "mysql_source", 
            "config": { 
                  "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector", 
                  "tasks.max": 1, 
                  "connection.url": "jdbc:mysql://mysql:3306/source", 
                  "connection.user": "root", 
                  "connection.password": "12345", 
                  "mode": "timestamp+incrementing", 
                  "incrementing.column.name": "id",
                  "timestamp.column.name": "updated_at", 
                  "catalog.pattern":"source",
                  "topic.prefix": "mysql-source-", 
                  "poll.interval.ms": 1000 } }' \
  http://localhost:8083/connectors
```

다음을 통해 정상적으로 진행됐는지 확인 가능

```jsx
curl -s -X GET http://localhost:8083/connectors/mysql_source/status | jq
```

KafDrop을 통한 모니터링 ({your_ip}:9000)

![Untitled (4)](https://user-images.githubusercontent.com/77625823/205200455-67805410-17a3-4a6d-a33f-7494294c2436.png)


PostgreSQL 타겟 DB 설정

```jsx
curl -X POST http://localhost:8083/connectors -H "Content-Type: application/json" \
   -d '{ 
        "name": "sink-pg",
        "config": {
            "connector.class":"io.confluent.connect.jdbc.JdbcSinkConnector",
            "tasks.max":1,
            "connection.url":"jdbc:postgresql://postgres:5432/target",
            "connection.user":"admin",
            "connection.password":"12345",
            "poll.interval.ms":"1000",
            "table.name.format": "${topic}",
            "topics.regex":"mysql-source-(.*)",
            "insert.mode":"upsert",
            "pk.mode":"record_value",
            "pk.fields":"id,created_at",
            "auto.create":"true",
            "auto.evolve":"true",
            "batch.size": 3000
        }
        }'
```

마찬가지로  다음 코드를 통해 정상적으로 Sink 됐는지 확인 가능

```jsx
curl -s -X GET http://localhost:8083/connectors/sink-pg/status | jq
```

![Untitled (5)](https://user-images.githubusercontent.com/77625823/205200479-53b3ef9c-44cb-4886-aa58-81eda4223593.png)


MongoDB 타겟 DB 설정

```jsx
**curl -X PUT http://localhost:8083/connectors/sink-mongodb-local/config -H "Content-Type: application/json" -d ' {
        "connector.class":"com.mongodb.kafka.connect.MongoSinkConnector",
        "tasks.max":"1",
        "topics":"mysql-source-first_table",
        "connection.uri":"mongodb://root:12345@mongodb:27017",
        "database":"kafka-test",
        "collection":"dev",
        "key.converter":"io.confluent.connect.avro.AvroConverter",
        "key.converter.schema.registry.url":"http://schema-registry:8081",
        "key.converter.schemas.enable":false,
        "value.converter":"io.confluent.connect.avro.AvroConverter",
        "value.converter.schema.registry.url":"http://schema-registry:8081"
}'**
```

```jsx
docker exec -it kafka-mongo mongosh --username root --password 12345 --authenticationDatabase admin

use kafka-test
db.dev.find()
```
