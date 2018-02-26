# btcpool矿池-nmcauxmaker模块解析

## nmcauxmaker命令使用

```shell
nmcauxmaker -c nmcauxmaker.cfg -l log_nmcauxmaker
#-c指定nmcauxmaker配置文件
#-l指定日志目录
```

## nmcauxmaker.cfg配置文件

```
nmcauxmaker = {
  //rpc调用间隔(秒)
  rpcinterval = 10;
  //最近一次rpc调用时间写入文件
  file_last_rpc_call_time = "/work/xxx/nmcauxmaker_lastrpccalltime.txt";
  //启动时是否检查zmq
  is_check_zmq = true;
  //nmc支付地址
  payout_address = "N59bssPo1MbK3khwPELTEomyzYbHLb59uY";
};

namecoind = {
  //nmc zmq地址和端口
  zmq_addr = "tcp://127.0.0.1:8331";
  //nmc rpc地址和端口
  rpc_addr    = "";  // http://127.0.0.1:8332
  //nmc rpc用户名和密码
  rpc_userpwd = "";  // username:password
};

//kafka集群
kafka = {
  brokers = "1.1.1.1:9092,2.2.2.2:9092,3.3.3.3:9092";
};

```

## namecoin-core安装

```
apt-get -y install build-essential libtool autotools-dev automake pkg-config libssl-dev libevent-dev bsdmainutils
apt-get -y install libboost-all-dev
add-apt-repository ppa:bitcoin/bitcoin
apt-get update
apt-get -y install libdb4.8-dev libdb4.8++-dev
apt-get -y install libqt5gui5 libqt5core5a libqt5dbus5 qttools5-dev qttools5-dev-tools libprotobuf-dev protobuf-compiler

cd /work/
wget https://github.com/namecoin/namecoin-core/archive/v0.12.0.tar.gz
tar -zxvf v0.12.0.tar.gz
cd namecoin-core-0.12.0/
./autogen.sh
./configure --with-incompatible-bdb --prefix=/work/namecoin
make
make install

cd /work/namecoin/bin/
./bitcoind --daemon -rpcuser=bitcoinrpc -rpcpassword=xxxx -testnet -zmqpubhashtx=tcp://0.0.0.0:18331 -zmqpubhashblock=tcp://0.0.0.0:18331
```

## nmcauxmaker流程图

待补充

## 参考文档

* [NameCoin compile from source on Ubuntu](http://www.reynoldtech.com/namecoin-compile-from-source-on-ubuntu/)