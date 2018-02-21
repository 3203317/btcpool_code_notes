# btcpool矿池-GbtMaker模块解析

### GbtMaker命令使用

GbtMaker用于从比特币节点获取挖矿模板，并发送给kafka。

```
gbtmaker -c gbtmaker.cfg -l log_dir
#-c指定gbtmaker配置文件
#-l指定日志目录
```

### gbtmaker.cfg配置文件

```
gbtmaker = {
  //rpc调用间隔(秒)
  rpcinterval = 5;
  //启动时是否检查zmq
  is_check_zmq = true;
};

bitcoind = {
  //指定zmq地址和端口
  zmq_addr = "tcp://127.0.0.1:8331";
  //指定rpc地址和端口，如http://127.0.0.1:8332
  rpc_addr    = "";
  //指定rpc账号和密码，格式如username:password
  rpc_userpwd = "";
};

kafka = {
  //指定kafka集群
  brokers = "1.1.1.1:9092,2.2.2.2:9092,3.3.3.3:9092";
};
```

### 挖矿模板范例

[getblocktemplate.txt](getblocktemplate.txt)

### GbtMaker流程图

![](GbtMaker.png)

### KAFKA_TOPIC_RAWGBT消息

```
return Strings::Format("{\"created_at_ts\":%u,"
	"\"block_template_base64\":\"%s\","
	"\"gbthash\":\"%s\"}",
	(uint32_t)time(nullptr), EncodeBase64(gbt).c_str(),
	gbtHash.ToString().c_str());
```

### 参考文档

* [Google-glog](http://www.xstring.cn/wiki/doku.php?id=apidoc:xlibrary:%E6%97%A5%E5%BF%97%E5%BA%93:glog)
* [librdkafka - the Apache Kafka C/C++ client library](https://github.com/edenhill/librdkafka)
* [ZeroMQ](https://github.com/zeromq/libzmq)
* [Bitcoin Core APIs](https://bitcoin.org/en/developer-reference#bitcoin-core-apis)
* [getblocktemplate](https://en.bitcoin.it/wiki/Getblocktemplate)