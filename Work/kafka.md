# 4.0.0

##### 下载 JDK（需要JRE61+）

```bash
cd /data/lib
wget https://download.oracle.com/java/24/latest/jdk-24_linux-x64_bin.tar.gz
tar -xvf jdk-24_linux-x64_bin.tar.gz
rm -f jdk-24_linux-x64_bin.tar.gz
vim /etc/profile

# add:
export JAVA_HOME=/data/lib/jdk-24.0.1
```



##### 下载 kafka

```bash
cd /data/lib

# need proxy
# check https://kafka.apache.org/downloads for new version
wget https://dlcdn.apache.org/kafka/4.0.0/kafka_2.13-4.0.0.tgz

tar -xvf kafka_2.13-4.0.0.tgz
rm -f kafka_2.13-4.0.0.tgz
cd kafka_2.13-4.0.0
```



##### 启动

Generate a Cluster UUID

```bash
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
```

Format Log Directories

```bash
bin/kafka-storage.sh format --standalone -t $KAFKA_CLUSTER_ID -c config/server.properties
```

修改返回给客户端的地址（客户端和kafka-server不在一起时必须设置；设置这个后 `/bin/kafka-console-consumer.sh` 将无法通过 localhost:9092 进行消费）

```bash
advertised.listeners=PLAINTEXT://xxx:9092
```



Start the Kafka Server

```bash
bin/kafka-server-start.sh config/server.properties
```



# 参考

https://kafka.apache.org/quickstart