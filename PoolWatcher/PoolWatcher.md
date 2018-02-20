# btcpool矿池-PoolWatcher模块解析

### PoolWatcher命令使用

PoolWatcher，用于监控第三方矿池并获取挖矿模板，发送给kafka。

```
poolwatcher -c poolwatcher.cfg -l log_poolwatcher
#-c指定poolwatcher配置文件
#-l指定日志目录
```

### poolwatcher.cfg配置文件

```
//是否使用testnet
testnet = true;

//第三方stratum server列表
pools = (
    //蚁池
    {
        name = "antpool";
        host = "stratum.antpool.com";
        port = 3333;
        worker = "worker.miner";
    },
    //鱼池
    {
        name = "f2pool";
        host = "stratum.f2pool.com";
        port = 3333;
        worker = "worker.miner";
    },
    //国池
    {
        name = "btcc";
        host = "stratum.btcchina.com";
        port = 3333;
        worker = "worker.miner";
    }
);

//kafka集群
kafka = {
    brokers = "1.1.1.1:9092,2.2.2.2:9092,3.3.3.3:9092";
};
```

### PoolWatcher流程图

![](GbtMaker.png)

### 参考文档

* [Libevent编程中文帮助文档](http://blog.csdn.net/zhouyongku/article/details/53431597/)
* [libevent库介绍--事件和数据缓冲](https://www.cnblogs.com/liunianshiwei/p/6059232.html)