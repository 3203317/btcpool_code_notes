# btcpool矿池-slparser(share log parser)模块解析

## 核心机制总结

* 支持三种功能：
	* 指定Date和UID，将打印指定日期指定用户的share信息到stdout
		* UID=0时，将打印指定日期所有用户的share信息
	* 指定Date但未指定UID，读取指定日期sharelog，统计数据并写入数据库
		* 按Worker、user、pool三个维度统计:Accept1h、Accept1d、score1h、score1d、Reject1h、Reject1d
		* 数据库仅保留最近3个月统计数据
	* 如果Date和UID均未指定，将监听文件变化，读取share并统计数据，每15秒写入数据库
		* 同时启动Httpd服务，开放ServerStatus和WorkerStatus

## slparser命令使用

```shell
slparser -c slparser.cfg -l log_dir
slparser -c slparser.cfg -l log_dir2 -d 20160830
slparser -c slparser.cfg -l log_dir3 -d 20160830 -u puid
#-c指定slparser配置文件
#-l指定日志目录
#-d指定日期
#-u指定PUID（即userId），userId为0时dump all, >0时仅输出指定userId的sharelog
```

## slparser.cfg配置文件

```shell
slparserhttpd = {
  #指定IP和端口
  ip = "0.0.0.0";
  port = 8081;

  #每间隔15s写库
  flush_db_interval = 15;
};

#指定sharelog文件路径
sharelog = {
  data_dir = "/data/sharelog";
};

#数据库配置，表为table.stats_xxxx
pooldb = {
  host = "";
  port = 3306;
  username = "dbusername";
  password = "dbpassword";
  dbname = "";
};
```

## slparser流程图

![](slparser.png)


## bpool_local_stats_db数据库结构

[bpool_local_stats_db.txt](bpool_local_stats_db.txt)

```shell
DROP TABLE IF EXISTS `stats_pool_day`;
CREATE TABLE `stats_pool_day` (
  `day` int(11) NOT NULL,
  `share_accept` bigint(20) NOT NULL DEFAULT '0',
  `share_reject` bigint(20) NOT NULL DEFAULT '0',
  `reject_rate` double NOT NULL DEFAULT '0',
  `score` decimal(35,25) NOT NULL DEFAULT '0.0000000000000000000000000',
  `earn` bigint(20) NOT NULL DEFAULT '0',
  `lucky` double NOT NULL DEFAULT '0',
  `created_at` timestamp NULL DEFAULT NULL,
  `updated_at` timestamp NULL DEFAULT NULL,
  UNIQUE KEY `day` (`day`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

DROP TABLE IF EXISTS `stats_pool_hour`;
CREATE TABLE `stats_pool_hour` (
  `hour` int(11) NOT NULL,
  `share_accept` bigint(20) NOT NULL DEFAULT '0',
  `share_reject` bigint(20) NOT NULL DEFAULT '0',
  `reject_rate` double NOT NULL DEFAULT '0',
  `score` decimal(35,25) NOT NULL DEFAULT '0.0000000000000000000000000',
  `earn` bigint(20) NOT NULL DEFAULT '0',
  `created_at` timestamp NULL DEFAULT NULL,
  `updated_at` timestamp NULL DEFAULT NULL,
  UNIQUE KEY `hour` (`hour`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

DROP TABLE IF EXISTS `stats_users_day`;
CREATE TABLE `stats_users_day` (
  `puid` int(11) NOT NULL,
  `day` int(11) NOT NULL,
  `share_accept` bigint(20) NOT NULL DEFAULT '0',
  `share_reject` bigint(20) NOT NULL DEFAULT '0',
  `reject_rate` double NOT NULL DEFAULT '0',
  `score` decimal(35,25) NOT NULL DEFAULT '0.0000000000000000000000000',
  `earn` bigint(20) NOT NULL DEFAULT '0',
  `created_at` timestamp NULL DEFAULT NULL,
  `updated_at` timestamp NULL DEFAULT NULL,
  UNIQUE KEY `puid_day` (`puid`,`day`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

DROP TABLE IF EXISTS `stats_users_hour`;
CREATE TABLE `stats_users_hour` (
  `puid` int(11) NOT NULL,
  `hour` int(11) NOT NULL,
  `share_accept` bigint(20) NOT NULL DEFAULT '0',
  `share_reject` bigint(20) NOT NULL DEFAULT '0',
  `reject_rate` double NOT NULL DEFAULT '0',
  `score` decimal(35,25) NOT NULL DEFAULT '0.0000000000000000000000000',
  `earn` bigint(20) NOT NULL DEFAULT '0',
  `created_at` timestamp NULL DEFAULT NULL,
  `updated_at` timestamp NULL DEFAULT NULL,
  UNIQUE KEY `puid_hour` (`puid`,`hour`),
  KEY `hour` (`hour`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

DROP TABLE IF EXISTS `stats_workers_day`;
CREATE TABLE `stats_workers_day` (
  `puid` int(11) NOT NULL,
  `worker_id` bigint(20) NOT NULL,
  `day` int(11) NOT NULL,
  `share_accept` bigint(20) NOT NULL DEFAULT '0',
  `share_reject` bigint(20) NOT NULL DEFAULT '0',
  `reject_rate` double NOT NULL DEFAULT '0',
  `score` decimal(35,25) NOT NULL DEFAULT '0.0000000000000000000000000',
  `earn` bigint(20) NOT NULL DEFAULT '0',
  `created_at` timestamp NULL DEFAULT NULL,
  `updated_at` timestamp NULL DEFAULT NULL,
  UNIQUE KEY `puid_worker_id_day` (`puid`,`worker_id`,`day`),
  KEY `day` (`day`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

DROP TABLE IF EXISTS `stats_workers_hour`;
CREATE TABLE `stats_workers_hour` (
  `puid` int(11) NOT NULL,
  `worker_id` bigint(20) NOT NULL,
  `hour` int(11) NOT NULL,
  `share_accept` bigint(20) NOT NULL DEFAULT '0',
  `share_reject` bigint(20) NOT NULL DEFAULT '0',
  `reject_rate` double NOT NULL DEFAULT '0',
  `score` decimal(35,25) NOT NULL DEFAULT '0.0000000000000000000000000',
  `earn` bigint(20) NOT NULL DEFAULT '0',
  `created_at` timestamp NULL DEFAULT NULL,
  `updated_at` timestamp NULL DEFAULT NULL,
  UNIQUE KEY `puid_worker_id_hour` (`puid`,`worker_id`,`hour`),
  KEY `hour` (`hour`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```