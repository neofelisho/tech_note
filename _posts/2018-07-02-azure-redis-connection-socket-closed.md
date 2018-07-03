---
layout      : post
title       : "Azure Redis Connection Exception: SocketClosed."
date        : 2018-07-02 23:00:00 +0800
categories  : programming
tags        : [Azure, Redis, RedisConnectionException, SocketClosed, Network Stability]
---

`Problem` There are several distributed webjobs in our system. Last weekend (late June 2018) there were many exceptions when our jobs tried to query data from specific one Redis server. But they could query other Redis servers normally.

![The inner exception message](https://c1.staticflickr.com/1/914/42263223555_22f7c7d524_o.png)

These jobs all followed the suggestion from this Azure doc: [Quickstart: Use Azure Redis Cache with a .NET application](https://docs.microsoft.com/en-us/azure/redis-cache/cache-dotnet-how-to-use-azure-redis-cache) to connect to Redis server.

```csharp
private static Lazy<ConnectionMultiplexer> lazyConnection = new Lazy<ConnectionMultiplexer>(() =>
{
    string cacheConnection = ConfigurationManager.AppSettings["CacheConnection"].ToString();
    return ConnectionMultiplexer.Connect(cacheConnection);
});

public static ConnectionMultiplexer Connection
{
    get
    {
        return lazyConnection.Value;
    }
}
```

`Solution` According to the comment of [this issue of StackExchange.Redis](https://github.com/StackExchange/StackExchange.Redis/issues/871), it's a network stability issue and will not be fixed in 1.x version of StackExchange.Redis.

Temporarily we changed the Redis connection code from Azure suggestion to traditional establishing, releasing connection by ourselves. And the problem was solved...or I should say the exceptions were disappeared. The codes looks like this:

```csharp
// TODO: Revert to lazy using after StackExchange.Redis fix the network stability issue.
// https://github.com/StackExchange/StackExchange.Redis/issues/871
string cacheConnection = ConfigurationManager.AppSettings["CacheConnection"].ToString();
var connection = ConnectionMultiplexer.Connect(cacheConnection);
try
{
    // Query something
}
catch (Exception e)
{
    Logger.Error($"Query something from Redis occurs problem.", e.InnerException);
}
finally
{
    connection?.Dispose();
}
```
