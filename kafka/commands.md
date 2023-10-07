# Kafka

## 增删改查

```bash
# topic list
kafka-topics.sh --list --zookeeper localhost:2181

# 查询 topic 详情
kafka-topics.sh --zookeeper localhost:2181 --describe --topic MARGIN_LOAN_ISOLATED_INTEREST_CHANGE

# 查看各个分区最大offset
./kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list 172.24.100.62:9092 --time -1 --topic SPOT_ACCOUNT_OPS_1
# 查看各个分区最小offset
./kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list 172.24.100.62:9092 --time -2 --topic SPOT_ACCOUNT_OPS_1

# 删除
bin/kafka-topics.sh --zookeeper zk_host:port --delete --topic TOPIC_NAME
```

## 重置消费者偏移量

`--to-earliest`：把位移调整到分区当前最小位移
`--to-latest`：把位移调整到分区当前最新位移
`--to-current`：把位移调整到分区当前位移
`--to-offset <offset>`： 把位移调整到指定位移处

```bash
# 重置指定消费组的指定主题偏移量  https://www.cnblogs.com/8765h/p/12233576.html
kafka-consumer-groups.sh --bootstrap-server 10.4.0.71:9092 --group trade_settlement --reset-offsets --topic KUMEX_ME_MATCH_XBTUSDTM --to-earliest --execute
# 重置该组所有主题的偏移量
kafka-consumer-groups.sh --bootstrap-server 10.4.0.71:9092 --group trade_settlement --reset-offsets --alltopics --to-earliest --execute
```

## 命令行启动消费者

```bash
# 自带消费者
export JAVA_HOME=/usr/local/jdk-11.0.12+7 && ./kafka-console-consumer.sh --bootstrap-server 172.24.16.101:9092 --from-beginning --topic platform-logs
./kafka-console-consumer.sh --bootstrap-server 172.24.16.101:9092 --offset latest --topic margin-logs
./kafka-console-consumer.sh --bootstrap-server 10.4.0.11:9092 --topic test

./kafka-console-producer.sh --broker-list 172.24.16.101:9092 --topic futures-logs

./kafka-console-consumer.sh --bootstrap-server 172.24.16.101:9092 --property print.timestamp=true --from-beginning --topic platform-logs
```

## 清空kfk

```bash
systemctl stop kafka
# 关闭kafka,删除索引，启动kafka会重建索引
find /data/kfk/data/kafka/kafka-logs/ -name "*.index" |xargs rm -f
find /data/kfk/data/kafka/kafka-logs/ -name "*.timeindex" |xargs rm -f
```

## 批量操作

### 批量修改配置

```bash
#!/bin/bash
# 600000 10m
# 1800000 0.5h
# 3600000 1h
# 86400000 1d

HOME_DIR="/opt/kafka-2.8.1/bin"

BOOTSTRAP_SERVER_PORT="9092"

IP=$(ifconfig eth0 |grep inet|grep -v 127.0.0.1|grep -v inet6|awk '{print $2}'|tr -d "addr:")

cd ${HOME_DIR}

./kafka-topics.sh --bootstrap-server=${IP}:${BOOTSTRAP_SERVER_PORT} --list > /tmp/topics.txt
sed -i 's/__consumer_offsets//g' /tmp/topics.txt
for topic in $(cat /tmp/topics.txt); do
  if [ ${topic} == '__consumer_offsets' ]; then
      echo "skip topic __consumer_offsets"
      continue
  fi
  echo ${topic}
  ./kafka-configs.sh --bootstrap-server ${IP}:${BOOTSTRAP_SERVER_PORT} --entity-type topics --entity-name ${topic} --alter --add-config='retention.ms=600000'
done
```

### 批量修改配置2.1.1版本

```bash
#!/bin/bash
# 600000 10m
# 1800000 0.5h
# 3600000 1h
# 86400000 1d

IP=$(ifconfig eth0 |grep inet|grep -v 127.0.0.1|grep -v inet6|awk '{print $2}'|tr -d "addr:")

HOME_DIR="/opt/kafka-2.1.1/bin"
ZOOKEEPER_PORT="2181"

cd ${HOME_DIR}

./kafka-topics.sh --zookeeper ${IP}:${ZOOKEEPER_PORT} --list > /tmp/topics.txt
sed -i 's/__consumer_offsets//g' /tmp/topics.txt
for topic in $(cat /tmp/topics.txt); do
  if [ ${topic} == '__consumer_offsets' ]; then
      echo "skip topic __consumer_offsets"
      continue
  fi
  echo ${topic}
  ./kafka-configs.sh --zookeeper ${IP}:${ZOOKEEPER_PORT} --entity-type topics --entity-name ${topic} --alter --add-config='retention.ms=3600000'
done
```

