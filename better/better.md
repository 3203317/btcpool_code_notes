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
* 三种情况下将立即向kafka发送StratumJob：
	* 高度大于本地高度（即已发现新块）
	* 高度与本地高度相同，但前个Job为空块Job，但新Gbt非空块
	* 达到预定的时间间隔20秒（可从配置文件中指定）

### ③sserver

* 接收的job延迟超过60秒将丢弃
* 如果job中prevHash与本地job中prevHash不同，即为已产生新块，job中isClean状态将置为true
	* true即要求矿机立即切换job
* 三种情况下将向矿机下发新job：
	* 收到新高度的job
	* 过去一个job为新高度且为空块job，且最新job为非空块job
	* 达到预定的时间间隔30秒
* 最近一次下发job的时间将写入文件（由file_last_notify_time指定）
* 本地job有效期为300秒
* 每10秒拉取一次新用户列表（由list_id_api_url指定），用户写入本地map中
* sserver最大可用SessionId数为16777214
* btcpool支持BtcAgent扩展协议和Stratum协议，使用magic_number（0x7F）区分
* 处理Stratum协议：
	* suggest_target与suggest_difficulty等价，用于设置初始挖矿难度，需在subscribe之前请求
	* 使用sessionID作为extraNonce1_以确保矿机任务不重复
	* authorize之前有15秒读超时，authorize之后有10分钟读超时，10分钟无提交将断开连接
	* 初始难度为16384，或从suggest_difficulty指定，下次将一次调整到位保持10s提交share
	* 每个session会维护一个localJobs_队列，队列长度为10条
* share被拒绝的几种情况：
	* JOB_NOT_FOUND，localJobs_队列中该job已被挤出
	* DUPLICATE_SHARE，share已提交过，已提交的share会计入submitShares_
	* JOB_NOT_FOUND，jobRepository_中job不存在，即job已过期（300秒过期时间）
	* JOB_NOT_FOUND，jobRepository_中job状态为Stale，即job是旧的非新job
	* TIME_TOO_OLD，share中提交的nTime小于job规定的minTime
	* TIME_TOO_OLD，share中提交的nTime比当前时间大10分钟
	* LOW_DIFFICULTY，share中提交的hash不满足难度目标
* 处理BtcAgent扩展协议：
	* Agent下矿机默认难度也为16384
	* 使用Agent sessionID作为extraNonce2_前半部分，以确保Agent下矿机任务不重复
	* 矿池下发新任务时，如session为BtcAgent，将为Agent下所有矿机计算难度
		* 如难度发生变更，将按难度不同，分别构造多条CMD_MINING_SET_DIFF指令一并下发处理

### ④blkmaker

* blkmaker可以连多个bitcoind节点
* blkmaker监听和接收4类消息：RAWGBT、STRATUM_JOB、SOLVED_SHARE和NMC_SOLVED_SHARE
* 监听RAWGBT目的为获取gbtHash/交易列表，用于构建Block，gbtHash和vtxs写入rawGbtMap_
	* rawGbtMap_保存最近100条gbtHash/vtxs对
* 监听STRATUM_JOB目的为获取jobId_/gbtHash，jobId_和gbtHash写入jobId2GbtHash_
	* jobId2GbtHash_保存最近120条jobId_/gbtHash对
* 监听SOLVED_SHARE目的为获取BlockHeader和coinbaseTx
	* BlockHeader+coinbaseTx+vtxs构造Block
* 构造好的Block会提交连接的所有bitcoind节点
* 构造好的Block入库，入库字段包括：
	* puid、worker_id、worker_full_name、job_id、height、hash
	* rewards（即coinbaseValue）、size（即blksize）、prev_hash、bits、version、created_at
	* created_at为入库时间非爆块时间
	
### ⑤sharelogger

* 接收SHARE_LOG，写入shares_，每2秒写入文件（路径由data_dir指定）
	* 每天一个新文件，文件名形如：sharelog-2016-07-12.bin
	* 最多维护最近3天的文件句柄

### ⑥slparser

* 支持三种功能：
	* 指定Date和UID，将打印指定日期指定用户的share信息到stdout
		* UID=0时，将打印指定日期所有用户的share信息
	* 指定Date但未指定UID，读取指定日期sharelog，统计数据并写入数据库
		* 按Worker、user、pool三个维度统计:Accept1h、Accept1d、score1h、score1d、Reject1h、Reject1d
		* 数据库仅保留最近3个月统计数据
	* 如果Date和UID均未指定，将监听文件变化，读取share并统计数据，每15秒写入数据库
		* 同时启动Httpd服务，开放ServerStatus和WorkerStatus

### ⑦statshttpd

### ⑧

### ⑨

## 优化思考