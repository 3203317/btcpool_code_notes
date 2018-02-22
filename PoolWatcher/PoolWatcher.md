# btcpool矿池-PoolWatcher模块解析

## 核心机制总结

* 监听StratumJob，更新poolStratumJob_，用于和第三方矿池比对
* 作为client连接第三方矿池，如收到挖矿任务，仅当接收的job高度=本地矿池job高度+1时，将构造EmptyGBT
* 如下几种情况将丢弃从第三方矿池接收的job：
	* job高度与本地矿池job高度相同
	* job高度不等于本地矿池job高度+1，高度跳跃太大
	* nBits与本地矿池job nBits不同

## PoolWatcher命令使用

PoolWatcher，用于监控第三方矿池并获取挖矿模板，发送给kafka。

```shell
poolwatcher -c poolwatcher.cfg -l log_poolwatcher
#-c指定poolwatcher配置文件
#-l指定日志目录
```

## poolwatcher.cfg配置文件

```shell
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

## PoolWatcher流程图

![](GbtMaker.png)

## 参考文档

* [Libevent编程中文帮助文档](http://blog.csdn.net/zhouyongku/article/details/53431597/)
* [libevent库介绍--事件和数据缓冲](https://www.cnblogs.com/liunianshiwei/p/6059232.html)