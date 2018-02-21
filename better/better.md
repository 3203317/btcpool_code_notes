# btcpool矿池-核心机制总结及优化思考

## 核心机制总结

### gbtmaker

* 监听Bitcoind ZMQ中BITCOIND_ZMQ_HASHBLOCK消息，一有新块产生，将立即向kafka发送新Gbt
* 另默认每5秒间隔（可从配置文件中指定）主动RPC请求Bitcoind，获取Gbt发送给kafka
* Gbt消息大小约2M，含交易列表

## GbtMaker优化点

* gbtmaker默认每5秒从bitcoind获取getblocktemplate，是否需加快频率???
* jobmaker中监听NMC域名币消息以实现混合挖矿，是否仍有价值???
* gbtmaker每5秒获取一次gbt，而jobmaker每20秒生成一次新job，时间设置是否合理???
* sserver除收到新高度job立即下发外，每30秒重新下发一次新job，时间设置是否合理???

