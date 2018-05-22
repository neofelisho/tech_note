---
layout      : post
title       : "Test the effect of Redis connection setting: syncTimeout."
date        : 2018-05-22 20:14:00 +0800
categories  : post
tags        : [azure, redis, timeout, syncTimeout]
---

**Story**
[As previous post](https://neofelisho.github.io/neofelisho.github.io/post/2018/05/19/diagnostic-timeout-exceptions-for-azure-redis.html) mentioned, there were many redis timeout exceptions in our system. After we altered the `minWorkerThread` setting, we eliminated most of the HTTP 500 exceptions. But we still had many HTTP 401 exceptions and there was no problem if we lower the system loading from 10000 ppls to 1000 ppls concurrently.

---

`Problem` Lack async

Our system uses Redis as OAuth token storage, it looks kind of what Spring Framework does. But our edition uses no async/await function when accessing Redis. I altered the Redis connection setting to make sure what the problem was.

```xml=
<add name="RedisConnection" connectionString="somewhere.redis.cache.windows.net:6380,password=somepassword,ssl=True,abortConnect=False,connectTimeout=30000,syncTimeout=10000" />
```

`Reference` [Configuration Options](https://stackexchange.github.io/StackExchange.Redis/Configuration.html#configuration-options), [Investigating timeout exceptions in StackExchange.Redis for Azure Redis Cache](https://azure.microsoft.com/zh-tw/blog/investigating-timeout-exceptions-in-stackexchange-redis-for-azure-redis-cache/)

`Testing Result`

syncTimeout=1000[syncTimeout=1000](https://c1.staticflickr.com/1/898/41375602575_cbfbccd8ba_o_d.png)

syncTimeout=5000[syncTimeout=5000](https://c1.staticflickr.com/1/951/40470558420_730575886f_o_d.png)

We got a little improvement...

---

`Problem` Computation bottleneck

Although it looked all ok from App Service Plan's monitor, but we still tested it. We increased the instance count from 10 to 18 (1.8x).

`Testing Result`

[syncTimeout=5000 and 1.8x intance counts](https://c1.staticflickr.com/1/965/42230890302_93accccc39_o_d.png)

--

`More Problems` After we completed these two tests, we found that the OAuth performance in our system is still too bad. Maybe there are some unknow bugs within it. I think I have to [continue the performance tuning](http://neofelisho.blogspot.tw/2018/04/performance-tuning-of-some-bad-codes.html) of our OAuth.

To be continued...