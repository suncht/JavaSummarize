```flow
st=>start: 浏览器/APP
httpService=>operation: 反向代理服务（Ngnix）
webService=>operation: Web服务器（Tomcat）
localService=>operation: 本地服务
rpcService=>operation: Rpc服务（Netty）
cacheService=>operation: Cache服务（Redis）
database=>operation: 数据库（MySQL）
useCache=>condition: 是否使用缓存?
useRpc=>condition: 是否调用RPC?
e=>end
st->httpService->webService->useRpc->useCache
useRpc(yes)->rpcService
useRpc(no)->localService
useCache(yes)->cacheService
useCache(no)->database
localService->useCache
rpcService->useCache
```

Ngnix QPS: 50w+
Tomcat QPS： <5K，一般3K
Redis QPS: 3W ~ 20W+，使用PipeLine，可达到10W+
Netty QPS：1W+
MySQL QPS：300 ~ 700

