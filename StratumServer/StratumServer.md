# btcpool矿池-StratumServer模块解析

## 核心机制总结

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

## StratumServer命令使用

```shell
sserver -c sserver.cfg -l log_dir
#-c指定sserver配置文件
#-l指定日志目录
```

## sserver.cfg配置文件

```shell
//是否使用testnet
testnet = true;

//kafka集群
kafka = {
  brokers = "1.1.1.1:9092,2.2.2.2:9092,3.3.3.3:9092";
};

//sserver配置
sserver = {
  //IP和端口
  ip = "0.0.0.0";
  port = 3333;

  //server id，全局唯一，取值范围[1, 255]
  id = 1;

  //最近一次挖矿通知时间写入文件，用于监控
  file_last_notify_time = "/work/xxx/sserver_lastnotifytime.txt";

  //如果启用模拟器，所有share均被接受，用于测试
  enable_simulator = false;

  //如果启用，所有share都将成块并被提交，用于测试
  enable_submit_invalid_block = false;

  //两次share提交的间隔时间
  share_avg_seconds = 10;
};

users = {
  //用户列表api
  list_id_api_url = "https://example.com/get_user_id_list";
};
```

## StratumServer流程图

* [Stratum mining protocol](StratumServer.png)

## 计算挖矿难度

```c++
//构造函数
//kMinDiff_为最小难度，static const uint64 kMinDiff_ = 64;
//kMaxDiff_为最大难度，static const uint64 kMaxDiff_ = 4611686018427387904ull;
//kDefaultDiff_为默认初始难度，static const uint64 kDefaultDiff_   = 16384;
//kDiffWindow_为N个share的时间窗口，static const time_t kDiffWindow_    = 900;
//kRecordSeconds_为1个share的时间，static const time_t kRecordSeconds_ = 10;
//sharesNum_和shares_初始值均为90
DiffController(const int32_t shareAvgSeconds) :
	startTime_(0),
	minDiff_(kMinDiff_), curDiff_(kDefaultDiff_), curHashRateLevel_(0),
	sharesNum_(kDiffWindow_/kRecordSeconds_), /* every N seconds as a record */
	shares_   (kDiffWindow_/kRecordSeconds_)
{
	if (shareAvgSeconds >= 1 && shareAvgSeconds <= 60) {
		shareAvgSeconds_ = shareAvgSeconds;
	} else {
		shareAvgSeconds_ = 8;
	}
}
//代码btcpool/src/StratumSession.h

//计算挖矿难度
//不低于最小难度64
uint64 DiffController::calcCurDiff() {
  uint64 diff = _calcCurDiff();
  if (diff < minDiff_) {
    diff = minDiff_;
  }
  return diff;
}

uint64 DiffController::_calcCurDiff() {
  const time_t now = time(nullptr);
  const int64 k = now / kRecordSeconds_;
  const double sharesCount = (double)sharesNum_.sum(k);
  if (startTime_ == 0) {  // first time, we set the start time
    startTime_ = time(nullptr);
  }

  const double kRateHigh = 1.40;
  const double kRateLow  = 0.40;
  //时间窗口（900秒）内预期的share数
  double expectedCount = round(kDiffWindow_ / (double)shareAvgSeconds_);

  //return now >= startTime_ + kDiffWindow_;
  if (isFullWindow(now)) { /* have a full window now */
    // big miner have big expected share count to make it looks more smooth.
    expectedCount *= minerCoefficient(now, k);
  }
  if (expectedCount > kDiffWindow_) {
    //最多1秒提交1个，预期share数最大为900
    expectedCount = kDiffWindow_;  // one second per share is enough
  }

  // this is for very low hashrate miner, eg. USB miners
  // should received at least one share every 60 seconds
  //非完整时间窗口、且时间已超过60s、提交的share数小于（60秒1个）、当前难度大于或等于2倍最小难度
  //此时降低难度为之前1/2
  if (!isFullWindow(now) && now >= startTime_ + 60 &&
      sharesCount <= (int32_t)((now - startTime_)/60.0) &&
      curDiff_ >= minDiff_*2) {
    setCurDiff(curDiff_ / 2);
    sharesNum_.mapMultiply(2.0);
    return curDiff_;
  }

  // too fast
  //如果提交share数超过预期数的1.4倍时
  if (sharesCount > expectedCount * kRateHigh) {
    //如果share数大于预期share数，且当前难度<最大难度时，提升难度为原来2倍
    while (sharesNum_.sum(k) > expectedCount && 
           curDiff_ < kMaxDiff_) {
      setCurDiff(curDiff_ * 2);
      sharesNum_.mapDivide(2.0); //share数/2
    }
    return curDiff_;
  }

  // too slow
  //如果是完整时间窗口，且当前难度大于或等于2被最小难度
  if (isFullWindow(now) && curDiff_ >= minDiff_*2) {
    //如果share数低于预期数的0.4，且当前难度大于或等于2倍最小难度，降低难度为原来的1/2
    while (sharesNum_.sum(k) < expectedCount * kRateLow &&
           curDiff_ >= minDiff_*2) {
      setCurDiff(curDiff_ / 2);
      sharesNum_.mapMultiply(2.0); //share数乘2
    }
    assert(curDiff_ >= minDiff_);
    return curDiff_;
  }
  
  return curDiff_;
}
```

## sserver校验share的机制

```shell
//本地job列表localJobs_最多保留最近10条任务，如有新任务，将挤出1条老任务。如果share所对应的job未在本地列表中，将StratumError::JOB_NOT_FOUND
//本地share列表submitShares_中，如果已有本条share，即重复，将StratumError::DUPLICATE_SHARE
//校验share不通过
//　　job列表exJobs_没有找到job，exJobs_中job有300秒过期时间，过期将删除，报StratumError::JOB_NOT_FOUND
//　　share中nTime小于job的最小时间，过老，报StratumError::TIME_TOO_OLD
//　　share中nTime超过job中的nTime 10分钟，过新，报StratumError::TIME_TOO_NEW
//　　区块哈希>job难度目标，不合格，报StratumError::LOW_DIFFICULTY
```

## sserver下发新job的机制

1、如果收到新高度statum job，将立即下发新job

```c++
bool isClean = false;
if (latestPrevBlockHash_ != sjob->prevHash_) {
	isClean = true;
	latestPrevBlockHash_ = sjob->prevHash_;
	LOG(INFO) << "received new height statum job, height: " << sjob->height_
	<< ", prevhash: " << sjob->prevHash_.ToString();
}
shared_ptr<StratumJobEx> exJob = std::make_shared<StratumJobEx>(sjob, isClean);
{
	ScopeLock sl(lock_);
    if (isClean) {
		// mark all jobs as stale, should do this before insert new job
		for (auto it : exJobs_) {
			it.second->markStale();
		}
	}
	// insert new job
	exJobs_[sjob->jobId_] = exJob;
}
if (isClean) {
	sendMiningNotify(exJob);
	return;
}
```

2、如果过去一个job为新高度且为空块job，并且最新job非空块job，将尽快下发新job

```
if (isClean == false && exJobs_.size() >= 2) {
	auto itr = exJobs_.rbegin();
	shared_ptr<StratumJobEx> exJob1 = itr->second;
	itr++;
	shared_ptr<StratumJobEx> exJob2 = itr->second;

	if (exJob2->isClean_ == true &&
		exJob2->sjob_->merkleBranch_.size() == 0 &&
		exJob1->sjob_->merkleBranch_.size() != 0) {
		sendMiningNotify(exJob);
	}
}
```

3、每超过一定时间间隔（30秒），将下发新job

```

void JobRepository::checkAndSendMiningNotify() {
	// last job is 'expried', send a new one
	if (exJobs_.size() &&
		lastJobSendTime_ + kMiningNotifyInterval_ <= time(nullptr))
	{
		shared_ptr<StratumJobEx> exJob = exJobs_.rbegin()->second;
		sendMiningNotify(exJob);
	}
}

JobRepository::JobRepository(const char *kafkaBrokers,
	const string &fileLastNotifyTime,
	Server *server):
running_(true),
kafkaConsumer_(kafkaBrokers, KAFKA_TOPIC_STRATUM_JOB, 0/*patition*/),
server_(server), fileLastNotifyTime_(fileLastNotifyTime),
kMaxJobsLifeTime_(300),
kMiningNotifyInterval_(30),  // TODO: make as config arg
lastJobSendTime_(0)
{
	assert(kMiningNotifyInterval_ < kMaxJobsLifeTime_);
}
```

## 参考文档

* [BtcAgent](https://github.com/btccom/btcagent)
* [BtcAgent通信协议](https://github.com/btccom/btcpool/blob/master/docs/AGENT.md)