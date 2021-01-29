## 下载安装



http://kafka.apache.org/

在tar包中，包含windows和linux的安装包 

## 基本使用

注意： Kafka依赖于Zookeeper，在开启Kafka之前需要先开启Zookeeper

​	

- 开启zookeeper

  需要修改对应的配置文件

  ![image-20201211163230906](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201211163230906.png)

  文件夹不需要提前创建

  ```powershell
  {KafkaDir}/bin/windows/zookeeper-server-start.bat ./config/zookeeper.properties
  ```

  

- 开启生产者

  ![image-20201211163439868](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201211163439868.png)

  ```powershell
  {KafkaDir}/bin/windows/kafka-server-start.bat ./config/server.properties
  ```

- 创建topic

  ```powershell
  {KafkaDir}/bin/windows/kafka-topics.bat --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions {partitionNum} --topic {topicName}
  ```

   参数说明：

  - --partitions	主题分区数目

  - --topic          主题名称

  - replication-factor  复制因子

    

  【注意】 在默认情况下，调用不存在的topic时，会自动创建topic



- 生产者发送消息

  ```powershell
  {KafkaDir}/bin/windows/kafka-console-producer.bat --broker-list localhost:9092 --topic test 
  > data1
  > data2
  > data3
  > ....
  ```

- 开启消费者并拉取消息

  ```powershell
  {KafkaDir}/bin/windows/kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic test \
  --from-beginning
  ```

  ![image-20201211165719531](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201211165719531.png)