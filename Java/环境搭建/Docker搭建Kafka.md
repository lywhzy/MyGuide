# Docker搭建Kafka

- 由于KAFKA基于Zookeeper，所以要拉取Zookeeper的镜像 docker pull wurstmeister/zookeeper
- 拉取KAFKA镜像 docker pull wurstmeister/kafka
- 根据镜像创建Zookeeper容器 docker run -d --name zookeeper --publish 2181:2181 --volume /etc/localtime:/etc/localtime zookeeper:latest
- 创建KAFKA容器 docker run -d --name kafka --publish 9092:9092 --link zookeeper --env KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 --env KAFKA_ADVERTISED_HOST_NAME=localhost --env KAFKA_ADVERTISED_PORT=9092 --volume /etc/localtime:/etc/localtime wurstmeister/kafka:latest