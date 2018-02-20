# btcpool bug总结

## 默认误启用WORK_WITH_STRATUM_SWITCHER并导致无法兼容标准Stratum协议

btcpool原意默认不启用WORK_WITH_STRATUM_SWITCHER。
但因CMakeLists.txt有一处bug，会导致默认误启用WORK_WITH_STRATUM_SWITCHER。

```
option(POOL__WORK_WITH_STRATUM_SWITCHER
"Work with Stratum Switcher" OFF)
```

此处代码折行，会导致POOL__WORK_WITH_STRATUM_SWITCHER使用默认值ON，而非代码中指定的OFF。
并进一步会导致如下代码有违原意。

```
if(POOL__WORK_WITH_STRATUM_SWITCHER)
  #代码有误会导致进入此地
  message("-- Work with Stratum Switcher (-DPOOL__WORK_WITH_STRATUM_SWITCHER=ON)")
  add_definitions(-DWORK_WITH_STRATUM_SWITCHER)
  set(POOL__DEB_PACKNAME_POSTFIX "-withswitcher")
else()
  #原意进入此地
  message("-- Work without Stratum Switcher (-DPOOL__WORK_WITH_STRATUM_SWITCHER=OFF)")
  set(POOL__DEB_PACKNAME_POSTFIX "")
endif()
```

反应到C++代码中即会导致无法兼容标准Stratum协议并报错，代码如下：
标准Stratum协议传入参数个数为1，形如：{"id": 17, "method": "mining.subscribe", "params": ["stratehm-stratum-proxy-0.8.1"]}。
如启用STRATUM_SWITCHER，要求参数个数>=2，形如：{"id": 1, "method": "mining.subscribe", "params": ["StratumSwitcher/0.1", "01ad557d", 203569230]}
因此代码会出错，导致矿机或agent收到响应： {"id":1,"result":null,"error":[27,"Illegal params",null]}并中断连接。

```
void StratumSession::handleRequest_Subscribe(const string &idStr,
                                             const JsonNode &jparams) {
  if (state_ != CONNECTED) {
    responseError(idStr, StratumError::UNKNOWN);
    return;
  }


#ifdef WORK_WITH_STRATUM_SWITCHER

  //
  // For working with StratumSwitcher, the ExtraNonce1 must be provided as param 2.
  // 
  //  params[0] = client version           [require]
  //  params[1] = session id / ExtraNonce1 [require]
  //  params[2] = miner's real IP (unit32) [optional]
  //
  //  StratumSwitcher request eg.:
  //  {"id": 1, "method": "mining.subscribe", "params": ["StratumSwitcher/0.1", "01ad557d", 203569230]}
  //  203569230 -> 12.34.56.78
  //

  //标准Stratum协议传入参数个数为1，形如：{"id": 17, "method": "mining.subscribe", "params": ["stratehm-stratum-proxy-0.8.1"]}
  //如启用STRATUM_SWITCHER，要求参数个数>=2，形如：{"id": 1, "method": "mining.subscribe", "params": ["StratumSwitcher/0.1", "01ad557d", 203569230]}
  //因此如下代码会出错，导致矿机或agent收到响应： {"id":1,"result":null,"error":[27,"Illegal params",null]}并中断连接
  if (jparams.children()->size() < 2) {
    responseError(idStr, StratumError::ILLEGAL_PARARMS);
    return;
  }

  //其他代码略
}
```


解决办法：
修改CMakeLists.txt，代码改为：

```
option(POOL__WORK_WITH_STRATUM_SWITCHER "Work with Stratum Switcher" OFF)
```

## 其他bug如有发现另行补充