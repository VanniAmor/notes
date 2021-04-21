按照Mongodb默认的配置，￼WiredTiger的写操作会先写入Cache，并持久化到WAL(Write ahead log)，每60s或log文件达到2GB时会做一次Checkpoint，将当前的数据持久化，产生一个新的快照。Wiredtiger连接初始化时，首先将数据恢复至最新的快照状态，然后根据WAL恢复数据，以保证存储可靠性。

![image-20210420210353422](https://raw.githubusercontent.com/VanniAmor/ImgBed/master/image-20210420210353422.png)



