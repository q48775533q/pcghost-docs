```bash
# jdk版本建议
https://github.com/apache/pulsar/blob/master/README.md#pulsar-runtime-java-version-recommendation

# 图省事，rpm安装 jdk17下载地址 
https://www.oracle.com/java/technologies/downloads/#java17
wget https://download.oracle.com/java/17/latest/jdk-17_linux-x64_bin.rpm

# pulsar下载地址，包括基础软件和各种连接器
https://pulsar.apache.org/download/

# pulsar-manager 
https://github.com/apache/pulsar-manager/tree/v0.4.0

wget https://archive.apache.org/dist/pulsar/pulsar-3.0.0/apache-pulsar-3.0.0-bin.tar.gz
wget https://dist.apache.org/repos/dist/release/pulsar/pulsar-manager/pulsar-manager-0.4.0/apache-pulsar-manager-0.4.0-bin.tar.gz

mkdir -p /data/soft/
cd /data/soft/
tar xvfz ~/apache-pulsar-3.0.0-bin.tar.gz

cd apache-pulsar-3.0.0

echo -e "export PULSAR_HOME=/data/soft/apache-pulsar-3.0.0\nexport PATH=\$PATH:\$PULSAR_HOME/bin" >> /etc/profile
mkdir -p /data/pulsar/{data/bookkeeper/journal,logs} /data/zookeeper/{data,logs}
source /etc/profile

# 连接器目录 参考 连接器下载参考下面
# mkdir /data/soft/apache-pulsar-3.0.0/connectors
# cd /data/soft/apache-pulsar-3.0.0/connectors
```

---

```bash

# 修改配置文件
sed  -i \
  -e 's/dataDir=data\/zookeeper/dataDir=\/data\/zookeeper\/data/g' \
  -e '$a server.1=192.168.3.101:2888:3888\nserver.2=192.168.3.102:2888:3888\nserver.3=192.168.3.103:2888:3888\n' \
  /data/soft/apache-pulsar-3.0.0/conf/zookeeper.conf 

# 修改zk的id，记得多台服务器不同
echo 3 > /data/zookeeper/data/myid 

# 启动zk 
PULSAR_EXTRA_OPTS="-Dstats_server_port=8001" pulsar-daemon start zookeeper 

# 初始化pulsar集群元数据到Zookeeper中
pulsar initialize-cluster-metadata \
  --cluster pulsar-cluster-prod \
  --zookeeper 192.168.3.101:2181 \
  --configuration-store 192.168.3.101:2181 \
  --web-service-url http://192.168.3.101:8080,192.168.3.102:8080,192.168.3.103:8080 \
  --broker-service-url pulsar://192.168.3.101:6650,192.168.3.102:6650,192.168.3.103:6650


# 部署Apache Bookkeeper

sed -i \
 -e 's/journalDirectory=data\/bookkeeper\/journal/journalDirectory=\/data\/pulsar\/data\/bookkeeper\/journal/g' \
 -e 's/ledgerDirectories=data\/bookkeeper\/ledgers/ledgerDirectories=\/data\/pulsar\/data\/bookkeeper\/ledgers/g' \
 -e 's/zkServers=localhost:2181/zkServers=192.168.3.101:2181,192.168.3.102:2181,192.168.3.103:2181/g' \
 /opt/pulsar/conf/bookkeeper.conf 

pulsar-daemon start bookie 

# 部署Broker
sed -i \
  -e 's/zookeeperServers=/zookeeperServers=192.168.3.101:2181,192.168.3.102:2181,192.168.3.103:2181/g' \
  -e 's/configurationStoreServers=/configurationStoreServers=192.168.3.101:2181,192.168.3.102:2181,192.168.3.103:2181/g' \
 -e 's/clusterName=/clusterName=pulsar-cluster-prod/g' \
 /opt/pulsar/conf/broker.conf
pulsar-daemon start broker

```

---

## 测试

```bash
1、配置客户端配置
sed -i \
  -e 's/webServiceUrl=http:\/\/localhost:8080\//webServiceUrl=http:\/\/192.168.3.101:8080,192.168.3.102:8080,192.168.3.103:8080\//g' \
  -e 's/brokerServiceUrl=pulsar:\/\/localhost:6650\//brokerServiceUrl=pulsar:\/\/192.168.3.101:6650,192.168.3.102:6650,192.168.3.103:6650\//g' \
  /opt/pulsar/conf/client.conf

2、配置命名空间的权限
pulsar-admin namespaces set-persistence -a 3 -e 3 -w 3 -r 3 public/default
pulsar-admin namespaces get-persistence public/default

3、创建消费者消费消息
pulsar-client consume persistent://public/default/test -n 100 -s "consumer-test" -t "Exclusive"

4、创建生产者产生消息
pulsar-client produce persistent://public/default/test -n 1 -m "Hello Pulsar"
```



---

## Pulsar Manager
Github：https://github.com/apache/pulsar-manager#access-pulsar-manager

以二进制方式安装为例，docker或k8s相关安装配置的参考：https://github.com/apache/pulsar-manager#access-pulsar-manager 和 https://github.com/apache/pulsar-manager/blob/master/src/README.md
```bash
1、下载安装
pulsar_manager_version=0.2.0
curl -s -#  https://dist.apache.org/repos/dist/release/pulsar/pulsar-manager/pulsar-manager-0.2.0/apache-pulsar-manager-$pulsar_manager_version-bin.tar.gz | tar zxvf - -C /tmp && \
tar -xvf /tmp/pulsar-manager/pulsar-manager.tar -C /opt && \
cp -r /tmp/pulsar-manager/dist /opt/pulsar-manager/ui && \
rm -rf /tmp/pulsar-manager
2、编辑配置文件
只修改/opt/pulsar-manager/application.properties的以下配置项，其他不用

# 开启Swagger
swagger.enabled=true
# 设置默认集群
default.environment.name=pulsar-cluster-prod
default.environment.service_url=http://127.0.0.1:8080
3、启动
nohup /opt/pulsar-manager/bin/pulsar-manager >/dev/null 2>&1 &
4、设置用户名密码
CSRF_TOKEN=$(curl http://localhost:7750/pulsar-manager/csrf-token) && echo $CSRF_TOKEN
curl \
   -H 'X-XSRF-TOKEN: $CSRF_TOKEN' \
   -H 'Cookie: XSRF-TOKEN=$CSRF_TOKEN;' \
   -H "Content-Type: application/json" \
   -X PUT http://localhost:7750/pulsar-manager/users/superuser \
   -d '{"name": "用户名", "password": "密码", "description": "Administrator", "email": "邮箱地址"}'
5、访问
http://pulsar-manager服务器地址:7750/ui/index.html

http://pulsar-manager服务器地址:7750/swagger-ui.html
```







连接器下载地址参考
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-aerospike-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-canal-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-cassandra-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-kafka-connect-adaptor-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-dynamodb-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-kinesis-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-debezium-mysql-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-debezium-postgres-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-debezium-mongodb-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-debezium-mssql-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-debezium-oracle-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-elastic-search-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-file-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-flume-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-hbase-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-hdfs2-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-hdfs3-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-http-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-influxdb-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-jdbc-clickhouse-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-jdbc-mariadb-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-jdbc-postgres-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-jdbc-sqlite-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-kafka-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-mongo-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-netty-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-nsq-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-rabbitmq-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-redis-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-solr-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-netty-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-data-generator-3.0.0.nar
wget https://dlcdn.apache.org/pulsar/pulsar-3.0.0/connectors/pulsar-io-twitter-3.0.0.nar

