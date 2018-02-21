# btcpool矿池-StratumServer模块解析

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

```
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

```
//本地job列表localJobs_最多保留最近10条任务，如有新任务，将挤出1条老任务。如果share所对应的job未在本地列表中，将StratumError::JOB_NOT_FOUND
//本地share列表submitShares_中，如果已有本条share，即重复，将StratumError::DUPLICATE_SHARE
//校验share不通过
//　　job列表exJobs_没有找到job，exJobs_中job有300秒过期时间，过期将删除，报StratumError::JOB_NOT_FOUND
//　　share中nTime小于job的最小时间，过老，报StratumError::TIME_TOO_OLD
//　　share中nTime超过job中的nTime 10分钟，过新，报StratumError::TIME_TOO_NEW
//　　区块哈希>job难度目标，不合格，报StratumError::LOW_DIFFICULTY
```

## 参考文档

待补充