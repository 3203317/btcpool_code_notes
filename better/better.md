# btcpool矿池-优化思考

## GbtMaker优化点

* gbtmaker默认每5秒从bitcoind获取getblocktemplate，是否需加快频率???
* jobmaker中监听NMC域名币消息以实现混合挖矿，是否仍有价值???
* gbtmaker每5秒获取一次gbt，而jobmaker每20秒生成一次新job，时间设置是否合理???
