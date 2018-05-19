---
layout      : post
title       : "Diagnostic timeout exceptions in StackExchange.Redis for Azure Redis Cache"
date        : 2018-05-19 20:51:00 +0800
categories  : post
tags        : [azure, redis, timeout, application insights]
---

# Diagnostic timeout exceptions in StackExchange.Redis for Azure Redis Cache
###### tags: `azure` `redis` `timeout` `application insights`

## Problems
![Picture-01](https://2.bp.blogspot.com/-vtXs8coCoXg/Wv_I92kcKjI/AAAAAAAAUaU/tKeRGJ_QT2YmBwJ-6EH2zSbmHfbayCkfwCLcBGAs/s1600/Diagnostic-timeout-exceptions-for-redis-01.png)
>Timeout performing HEXISTS XXX.YYY:OAuthTokenStorages:AccessToken, inst: 267, mgr: Inactive, err: never, queue: 0, qu: 0, qs: 0, qc: 0, wr: 0, wq: 0, in: 0, ar: 0, clientName: RD__________F9, serverEndpoint: Unspecified/xxx.redis.cache.windows.net:6380, keyHashSlot: 1758, IOCP: (Busy=0,Free=1000,Min=1,Max=1000), WORKER: (Busy=45,Free=32722,Min=1,Max=32767) (Please take a look at this article for some common client-side issues that can cause timeouts: http://stackexchange.github.io/StackExchange.Redis/Timeouts) 

![Picture-02](https://4.bp.blogspot.com/-S4H1S4cWlcA/Wv_JcxvoYyI/AAAAAAAAUac/XBr29445ZcoclhYRukjB4bH-rFKMAC-kACLcBGAs/s1600/Diagnostic-timeout-exceptions-for-redis-02.png)
>Timeout performing ZRANGEBYSCORE issuenumber:1, inst: 1, mgr: Inactive, err: never, queue: 4, qu: 0, qs: 4, qc: 0, wr: 0, wq: 0, in: 287, ar: 0, clientName: RD00155DB10567, serverEndpoint: Unspecified/jsti-webapi-redis.redis.cache.windows.net:6380, keyHashSlot: 5222, IOCP: (Busy=0,Free=1000,Min=1,Max=1000), WORKER: (Busy=12,Free=32755,Min=1,Max=32767) (Please take a look at this article for some common client-side issues that can cause timeouts: http://stackexchange.github.io/StackExchange.Redis/Timeouts)
>
---

`Problem`: **Singleton of Redis Multiplextor?**

During the load test, we kept monitoringthe statistics of Redis, App Service Plan and VM of MS-SQL. The CPU, memory and disk I/O loading seemed to be ok. So maybe there are something wrong with network, or with our codes to manipulate Redis.

I added logs in constructor of Redis multiplextor to make sure that we kept singleton in each web sites. It passed.

---

`Problem`: **Too many busy worker threads**

[According to this section](https://gist.github.com/JonCole/db0e90bedeb3fc4823c2#burst-of-traffic): Burst of traffic: Notice that in the "IOCP" section and the "WORKER" section you have a "Busy" value that is greater than the "Min" value. This means that your threadpool settings need adjusting.

[And according to JonCole's another post about ThreadPool](https://gist.github.com/JonCole/e65411214030f0d823cb): in my first exception, there are 45 busy worker threads now and our system is configured to allow 1 minimum worker thread. So our system would cause (45-1)*500ms = 22 seconds delay...OMG.

`Resolution`: Currently we use P1 pricing tier of WebApp. It's single core with 1.75GB RAM. So maybe we should increase the "minIoThreads" and "minWorkerThreads" to 100 or more in configuration settings.

---

`Problem`: **Lack async**

Obviously, but I have to do more test about the effect between sync and async in Redis utilization.

---

`Mysterious`: BTW, the default settings of "minIoThreads" and "minWorkerThreads" are 4. Why the settings in our system became 1? 

---

## Reference
[https://stackexchange.github.io/StackExchange.Redis/Timeouts](https://stackexchange.github.io/StackExchange.Redis/Timeouts)

[Investigating timeout exceptions in StackExchange.Redis for Azure Redis Cache](https://azure.microsoft.com/zh-tw/blog/investigating-timeout-exceptions-in-stackexchange-redis-for-azure-redis-cache/)

[Diagnosing Redis errors on the client side](https://gist.github.com/JonCole/db0e90bedeb3fc4823c2)