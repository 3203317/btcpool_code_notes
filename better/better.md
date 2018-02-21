# btcpool矿池-核心机制总结及优化思考

## 核心机制总结

### ①gbtmaker

* 监听Bitcoind ZMQ中BITCOIND_ZMQ_HASHBLOCK消息，一有新块产生，将立即向kafka发送新Gbt
* 另默认每5秒间隔（可从配置文件中指定）主动RPC请求Bitcoind，获取Gbt发送给kafka
* Gbt消息大小约2M，含交易列表

### ②jobmaker

* 同时监听kafka KAFKA_TOPIC_RAWGBT和KAFKA_TOPIC_NMC_AUXBLOCK，以支持混合挖矿
* 接收的Gbt消息，如果与本地时间延迟超过60秒将丢弃，如果延迟超过3秒将打印log
* 可用的Gbt消息，将以gbtTime+isEmptyBlock+height来构造key写入本地Map，另gbtHash也会写入本地队列
* 本地gbtHash队列仅保存最近20条，本地gbtMap中Gbt消息有效期：非空Gbt有效期90秒，空Gbt有效期15秒，过期将清除
	* 有效期可从配置文件中指定
* Gbt消息如果高度低于本地Gbt高度，且本地Gbt非空，且与本地时间间隔没超过2倍stratumJobInterval_，Gbt消息将丢弃
* 三种情况下将向kafka发送StratumJob：
	* 1、高度大于本地高度（即已发现新块）
	* 2、高度与本地高度相同，但前个Job为空块Job，但新Gbt非空块
	* 3、达到预定的时间间隔20秒（可从配置文件中指定）

### ③sserver

### ④⑤⑥⑦⑧⑨

## GbtMaker优化点

* gbtmaker默认每5秒从bitcoind获取getblocktemplate，是否需加快频率???
* jobmaker中监听NMC域名币消息以实现混合挖矿，是否仍有价值???
* gbtmaker每5秒获取一次gbt，而jobmaker每20秒生成一次新job，时间设置是否合理???
* sserver除收到新高度job立即下发外，每30秒重新下发一次新job，时间设置是否合理???

