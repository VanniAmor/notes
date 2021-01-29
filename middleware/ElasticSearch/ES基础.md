## 安装与启动



https://www.elastic.co/cn/downloads/elasticsearch

注意： ES暂时不要使用最新版本，建议按照SpringBoot的pom中的es版本

**使用6.4.3版本**

https://www.elastic.co/cn/downloads/past-releases/elasticsearch-6-4-3

![image-20201217112649685](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201217112649685.png)



**修改elasticsearch.yml配置**

- cluster.name： 集群名称
- path.data： es数据存储路径，若路径不存在，则会自动创建
- path.log:     es日志存储路径， 若路径不存在，则会自动创建



**添加到环境变量**

![image-20201217160419232](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201217160419232.png)

**中文分词插件 ElasticSearch-Analysis-Ik**

https://github.com/medcl/elasticsearch-analysis-ik/releases/tag/v6.4.3

![image-20201217161923195](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201217161923195.png)

​		可以通过配置IKAnalyzer.xml文件，来额外拓展分词字典

**启动ES**

![image-20201217162123132](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201217162123132.png)

运行图中脚本文件即可

ES**默认使用端口是9200**

可以使用``curl -X GET "localhost:9200/_cat/health?v"``来查看是否启动成功

ps：需要使用cmd来启动curl命令，使用powershell无法调用curl命令

![image-20201217164523678](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201217164523678.png)

可以使用`curl -X GET "localhost:9200/_cat/nodes?v"`来查看集群中的es节点

![image-20201217165735165](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201217165735165.png)



## 索引相关

可以使用curl命令或postman工具来进行

- 查看当前索引

  `curl -X GET "localhost:9200/_cat/indices?v"`

  ![image-20201217170532842](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201217170532842.png)

- 新增索引

  `curl -X PUT "localhost:9200/test"`

  ![image-20201217171905124](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201217171905124.png)

  ​	因为新的索引没有指定分片，所以状态为yello

  ​	![image-20201217173033503](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201217173033503.png)



- 删除索引

  `curl -X DELETE "localhost:9200/test"`

  ![image-20201217172149008](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201217172149008.png)



## 内容相关

- 新增内容

  `curl -X PUT "localhost:9200/test/_doc/1"`

  即： {host:port}/{indexName}/_doc/{id}

  ![image-20201217173831886](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201217173831886.png)

- 查询数据(通过id来查询)

  `curl -X GET "localhost:9200/test/_doc/1"`

  即： {host:port}/{indexName}/_doc/{id}

  ![image-20201217174008854](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201217174008854.png)

  

- 删除数据

  `curl -X DELETE "localhost:9200/test/_doc/1"`

  