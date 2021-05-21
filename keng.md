## zookeeper

### 创建网络
```bash
docker network create --driver bridge --subnet 172.23.0.0/24 --gateway 172.23.0.1  zookeeper_network
```

- subnet 为一个子网掩码
172.23.0.0/24 网段为 172.23.0.1 - 172.23.0.255

- gateway 子网的网关

此处创建一个给 zookeeper 和 kafka 共用的网络，子网段随意只要不和已有网络冲突即可

### docker-compose 配置

```yml
version: '3.4'

services:
  zoo1:
    image: wurstmeister/zookeeper:latest
    restart: always
    hostname: zoo1
    container_name: zoo1
    ports:
      - 2181:2181
    volumes:
      - "./zoo1/data:/data"
      - "./zoo1/datalog:/datalog"
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=0.0.0.0:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888
    networks:
      default:
        ipv4_address: 172.23.0.11

  zoo2:
    image: wurstmeister/zookeeper:latest
    restart: always
    hostname: zoo2
    container_name: zoo2
    ports:
      - 2180:2181 # docker 2181 端口绑定到宿主机 2180
    volumes:
      - "./zoo2/data:/data"
      - "./zoo2/datalog:/datalog"
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=0.0.0.0:2888:3888 server.3=zoo3:2888:3888
    networks:
      default:
        ipv4_address: 172.23.0.12

  zoo3:
    image: wurstmeister/zookeeper:latest
    restart: always
    hostname: zoo3
    container_name: zoo3
    ports:
      - 2179:2181
    volumes:
      - "./zoo3/data:/data"
      - "./zoo3/datalog:/datalog"
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=0.0.0.0:2888:3888
    networks:
      default:
        ipv4_address: 172.23.0.13

networks:
  default:
    external:
      name: zookeeper_network
```

## Kafka

### docker-compose 配置

```yml
version: '2'

services:
  kafka1:
    image: wurstmeister/kafka
    restart: always
    hostname: kafka1
    container_name: kafka1
    ports:
    - "9092:9092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://192.168.88.21:9092 # 宿主机 IP
      KAFKA_ADVERTISED_HOST_NAME: 192.168.88.21 # 宿主机 IP
      KAFKA_ADVERTISED_PORT: 9092
      KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
      JMX_PORT: 6501
    volumes:
    - ./kafka1/logs:/kafka # 绑定 docker 内文件夹 /kafka 到宿主机 ./kafka1/logs
    external_links:
    - zoo1
    - zoo2
    - zoo3
    networks:
      default:
        ipv4_address: 172.23.0.14

  kafka2:
    image: wurstmeister/kafka
    restart: always
    hostname: kafka2
    container_name: kafka2
    ports:
    - "9093:9092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://192.168.88.21:9093 
      KAFKA_ADVERTISED_HOST_NAME: 192.168.88.21
      KAFKA_ADVERTISED_PORT: 9093
      KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
      JMX_PORT: 6502
    volumes:
    - ./kafka2/logs:/kafka
    external_links:  # 连接本 compose 文件以外的 container
    - zoo1
    - zoo2
    - zoo3
    networks:
      default:
        ipv4_address: 172.23.0.15

  kafka3:
    image: wurstmeister/kafka
    restart: always
    hostname: kafka3
    container_name: kafka3
    ports:
    - "9094:9092"
    environment:
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://192.168.88.21:9094
      KAFKA_ADVERTISED_HOST_NAME: 192.168.88.21
      KAFKA_ADVERTISED_PORT: 9094
      KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
      JMX_PORT: 6503
    volumes:
    - ./kafka3/logs:/kafka
    external_links:  # 连接本 compose 文件以外的 container
    - zoo1
    - zoo2
    - zoo3
    networks:
      default:
        ipv4_address: 172.23.0.16

  kafka-manager:
    image: sheepkiller/kafka-manager:latest
    restart: always
    container_name: kafka-manager
    hostname: kafka-manager
    ports:
      - "9002:9000"
    links:            # 连接本compose文件创建的container
      - kafka1
      - kafka2
      - kafka3
    external_links:   # 连接本compose文件以外的container
      - zoo1
      - zoo2
      - zoo3
    environment:
      ZK_HOSTS: zoo1:2181,zoo2:2181,zoo3:2181
      KAFKA_BROKERS: kafka1:9092,kafka2:9093,kafka3:9094
      APPLICATION_SECRET: letmein
      KM_ARGS: -Djava.net.preferIPv4Stack=true
    networks:
      default:
        ipv4_address: 172.23.0.10

networks:
  default:
    external:   # 和 zookeeper 使用同一个网络
      name: zookeeper_network
```

### 问题

1. kafka 中的 meta.properties 每次重启需删除
```bash
sudo rm -r ./kafka1
sudo rm -r ./kafka2
sudo rm -r ./kafka3
```

2. 开放端口

所有 kafka 端口都需要开放，否则 kafka 节点相互连不上，会报错

```bash
# 开放 9092 端口
firewall-cmd --zone=public --add-port=9092/tcp --permanent
# 重新载入以使开放的端口生效
firewall-cmd --reload
# 检查是否开放
firewall-cmd --zone=public --query-port=9092/tcp
```

3. 测试时需要添加一些配置

```bash
docker exec -it kafka1 /bin/sh
```

/opt/kafka_2.11-2.0.0/bin

详见 [PR](https://github.com/apache/kafka/pull/1983/commits/2c5d40e946bcc149b1a9b2c01eced4ae47a734c5)

## MySQL 

### docker-compose 配置

```yml
version: '3.1'

services:

  mysql:
    image: mysql:5.7
    container_name: mysql-5.7
    restart: always
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=000000
    volumes:
      # - ./mysql/mysqld.cnf:/etc/mysql/mysql.conf.d/mysqld.cnf
      - ./mysql/my.cnf:/etc/mysql/my.cnf # 使用本地 my.cnf
      - ./var/lib/mysql:/var/lib/mysql
      - ./mysql/init/:/docker-entrypoint-initdb.d/
    networks:
      default:
        ipv4_address: 172.26.0.17 # MySQL host

networks:
  default:
    external:
      name: mysql_network
```

不配置 ipv4_address 的情况下需要通过 `docker inspect mysql-5.7` 查询 mysql host IP，详见 [ISSUES](https://github.com/zendesk/maxwell/issues/1280)

### my.cnf
```cnf
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/

[mysqld]
server_id=1
log-bin=master
binlog_format=row
```

### 开放端口

开放端口，否则连接不上

```bash
# 开放 3306 端口
firewall-cmd --zone=public --add-port=3306/tcp --permanent
# 重新载入以使开放的端口生效
firewall-cmd --reload
# 检查是否开放
firewall-cmd --zone=public --query-port=3306/tcp
```

### 创建用户并添加权限

```bash
CREATE USER 'debezium'@'%' IDENTIFIED BY 'pacman';
GRANT ALL ON cornerstone.* TO 'debezium'@'%';
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'debezium'@'%';
FLUSH PRIVILEGES;
```

## Maxwell

```bash
docker run -it --rm zendesk/maxwell bin/maxwell --user='maxwell' --password='maxwell' --host='47.112.200.141' --producer=kafka --kafka.bootstrap.servers=172.23.0.14:9092 --kafka_topic=maxwell --log=debug
```

### docker-compose

```yml
version: '2'

services:
  maxwell:
    container_name: maxwell
    image: zendesk/maxwell
    command: ./bin/maxwell --user='maxwell' --password='maxwell' --host='47.112.200.141' --producer=kafka --kafka.bootstrap.servers=172.23.0.14:9092 --kafka_topic=maxwell --log=debug

networks:
  default:
    external:   # 使用已创建的网络
      name: zookeeper_network
```

## kafka connect

### docker-compose

```yml
version: '2'

services:
  connect:
    image: debezium/connect:1.2
    restart: always
    hostname: connect
    container_name: connect
    ports:
    - "8083:8083"
    environment:
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: my_connect_configs
      OFFSET_STORAGE_TOPIC: my_connect_offsets
      STATUS_STORAGE_TOPIC: my_connect_statuses
      BOOTSTRAP_SERVERS: kafka1:9092
    volumes:
    - ./logs:/kafka/logs
    external_links:
    - zoo1
    - zoo2
    - zoo3
    - kafka1
    - kafka2
    - kafka3
    - mysql-5.7
    networks:
      default:
        ipv4_address: 172.23.0.21

networks:
  default:
    external:   # 使用已创建的网络
      name: zookeeper_network
```

### 添加用户

```bash
CREATE USER 'debezium'@'%' IDENTIFIED WITH mysql_native_password BY 'pacman'; # 此处天坑
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'debezium'@'%';
GRANT ALL PRIVILEGES ON cornerstone_beta.* TO 'debezium'@'%';
FLUSH PRIVILEGES;
```

**mysql8 默认使用 sha256_password 方式身份验证，这种验证方式需要在数据库连接上添加 allowPublicKeyRetrieval=true 参数，而 debezium 没有添加，所以连接数据库时会报错。所以在使用 mysql8 时我们需要指定使用 mysql_native_password 方式身份验证**

### 注册 connector

- 注册
```bash
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" 172.23.0.21:8083/connectors/ -d '{ "name": "inventory-connector", "config": { "connector.class": "io.debezium.connector.mysql.MySqlConnector", "tasks.max": "1", "database.hostname": "mysql-5.7", "database.port": "3306", "database.user": "maxwell", "database.password": "maxwell", "database.server.id": "184054", "database.server.name": "dbserver1", "database.whitelist": "maxwell", "database.history.kafka.bootstrap.servers": "kafka1:9092", "database.history.kafka.topic": "dbhistory.inventory" } }'
```

- 检查是否注册成功
```bash
curl -H "Accept:application/json" 172.23.0.21:8083/connectors/
```

```bash
curl -H "Accept:application/json" 192.168.88.123:8093/connectors/yzl_pevc/status
```

```bash
curl -i -X POST 192.168.88.123:8093/connectors/pevc_yzl_v5/tasks/0/restart
```

- 开发站
```bash
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" 172.23.0.21:8083/connectors/ -d '{
    "name": "cornerstone_beta",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "tasks.max": "1",
        "database.hostname": "47.112.200.141",
        "database.port": "3306",
        "database.user": "debezium",
        "database.password": "pacman",
        "database.server.id": "184054",
        "database.server.name": "cornerstone",
        "database.whitelist": "cornerstone_beta",
        "database.history.kafka.bootstrap.servers": "kafka1:9092",
        "database.history.kafka.topic": "cornerstone_beta",
        "key.converter": "io.confluent.connect.avro.AvroConverter",
        "value.converter": "io.confluent.connect.avro.AvroConverter",
        "key.converter.schema.registry.url": "http://schema-registry:8081",
        "value.converter.schema.registry.url": "http://schema-registry:8081"
    }
}'
```


- 开发站
```bash
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" 192.168.88.123:8093/connectors/ -d '{
    "name": "pevc_yzl_v5",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "tasks.max": "1",
        "database.hostname": "192.168.88.11",
        "database.port": "3306",
        "database.user": "debezium",
        "database.password": "pacman",
        "database.server.id": "2021052101",
        "database.server.name": "pevc_yzl_v5",
        "database.whitelist": "pevc",
        "database.history.kafka.bootstrap.servers": "kafka1:9092",
        "database.history.kafka.topic": "pevc_yzl_v5_history",
        "snapshot.mode":"schema_only",
        "snapshot.locking.mode":"none"
    }
}'
```

curl -X DELETE http://localhost:8083/connectors/<connector-name>

binlog-do-db 需要打开

- 产品库
```bash
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" 172.33.0.21:8083/connectors/ -d '{
    "name": "pevc",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "tasks.max": "1",
        "database.hostname": "47.112.200.141",
        "database.port": "3306",
        "database.user": "debezium",
        "database.password": "pacman",
        "database.server.id": "184056",
        "database.server.name": "pevc",
        "database.whitelist": "pevc",
        "database.history.kafka.bootstrap.servers": "kafka1:9092",
        "database.history.kafka.topic": "pevc_beta"
    }
}'
```


```bash
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" 172.33.0.21:8083/connectors/ -d '{
    "name": "pevc",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "tasks.max": "1",
        "database.hostname": "10.20.70.21",
        "database.port": "3306",
        "database.user": "debezium",
        "database.password": "pacman",
        "database.server.id": "184056",
        "database.server.name": "pevc",
        "database.whitelist": "pevc",
        "database.history.kafka.bootstrap.servers": "kafka1:9092",
        "database.history.kafka.topic": "pevc_beta"
    }
}'
```


```
curl -X GET http://localhost:8081/subjects/cornerstone.cornerstone_beta.customers-value/versions/1
```





curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" 10.20.70.22:8085/connectors/ -d '{
"name": "datapack_connector",
"config": {
"connector.class": "io.debezium.connector.mysql.MySqlConnector",
"tasks.max": "1",
"database.hostname": "10.20.70.21",
"database.port": "3306",
"database.user": "dp_delta_beta",
"database.password": "pacman",
"database.server.id": "2021040106",
"database.server.name": "datapack_connector_server",
"database.whitelist": "datapack_enterprise,datapack_institution,datapack_deals,datapack_region,datapack_corporate_group",
"database.history.kafka.bootstrap.servers": "kafka1_datapack_delta:9092",
"database.history.kafka.topic": "datapack_connector_history",
"snapshot.mode":"schema_only",
"snapshot.locking.mode":"none",
"handling.mode":"warn"
}
}'


connect restful api https://docs.confluent.io/platform/current/connect/references/restapi.html



监控 connect https://debezium.io/documentation/reference/1.4/operations/monitoring.html