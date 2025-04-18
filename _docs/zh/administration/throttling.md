---
permalink: administration/throttling
---

# 关于流控
流量控制是指通过一些手段来控制读写请求的速度。

为什么要做流量控制？主要是减小集群压力，提升稳定性。如果集群的写流量太大，就会消耗大量的系统资源（CPU、IO等），从而影响读请求的延迟。有些业务对读性能要求比较高，如果对写流量不加控制，就无法保证服务质量。

从流控的作用位置来看，可分为：
* 客户端流控：从源头掐住流量。优点是避免不必要的网络传输；缺点是需要在客户端增加逻辑，且因为无法掌控用户的使用方式造成流控难以准确。
* 服务端流控：在ReplicaServer节点上进行流控。优点是对客户端透明，流控集中容易掌控；缺点是只能通过增大延迟或者拒绝请求的方式来流控，不够直接，另外可能无法避免不必要的网络传输。

从流控的粒度来看，可分为：
* 表级流控：只控制单个表的流控，粒度较细。
* 节点级流控：针对ReplicaServer节点进行的流控，不区分具体的表（节点级流控暂未支持）。

# 客户端流控

目前Java客户端提供了流控工具，参见[Java客户端文档#流量控制](/clients/java-client#流量控制)。

# 服务端流控

## 表写级流控

从`v1.11.2`版本开始，Pegasus增加了Server端基于写QPS的表级流控。从`v1.12.0`版本开始，Pegasus增加了基于吞吐量的表级流控。

实现原理：
* 用户可以在[Table环境变量](table-env)中设置`replica.write_throttling`和`replica.write_throttling_by_size`环境变量。其中`replica.write_throttling`是基于qps的限流，`replica.write_throttling_by_size`是基于吞吐量的限流。
* MetaServer将环境变量异步地通知到各个ReplicaServer，使该表的每个replica都获取到该环境变量，这个过程大约有几秒到几十秒不等的延迟，但是不会超过一分钟。
* replica获得环境变量后，解析获得write_throttling流控配置，并立即开始生效。

write_throttling流控目前支持两种操作类型：
* delay：server端收到请求后不立即处理，而是推迟一段时间后再处理，这样使client端的写延迟增大，间接达到流控的目的。
* reject：server端收到请求后不进行处理，而是返回ERR_BUSY的错误码。可以推迟一段时间再返回错误码，以增大客户端收到错误的延迟，避免客户端立即重试，造成频繁的不必要重试。

环境变量`replica.write_throttling`/`replica.write_throttling_by_size`的value格式：
```
{delay_qps_threshold}*delay*{delay_ms},{reject_qps_threshold}*reject*{delay_ms_before_reject}
```
注：
* delay_qps_threshold：触发delay操作的QPS阈值。如果1秒内处理的写请求数超过这个值，则这1秒内后面的请求都执行delay操作。
* delay_ms：delay操作的推迟时间，单位毫秒，需满足>=0。
* reject_qps_threshold：触发reject操作的QPS阈值。如果1秒内处理的写请求数超过这个值，则这1秒内后面的请求都执行reject操作。
* delay_ms_before_reject：reject操作返回错误码之前的推迟时间，单位毫秒，需满足>=0。
* delay和reject配置可以同时提供两个，也可以只提供其中一个。
* 如果delay和reject配置同时提供，且QPS同时达到了delay和reject的阈值，那么会执行reject操作。
* 表级流控会均摊到每个分片上，因此不宜设置太小。假设`temp`表具有256个分片，并设置下方示例中的流控参数，具体流控判定时，若某分片QPS超过（1000/256）则将采取delay操作，某分片QPS超过（2000/256）将采取reject操作。若某分片流量超过（10^9/256）`字节/秒`则将采取delay操作，若某分片流量超过（20^9/256）`字节/秒`则将采取reject操作。

**写流控设置示例：**

```bash
$ ./run.sh shell -n {clusterName}
>>> use temp
OK
>>> set_app_envs replica.write_throttling 1000*delay*100,2000*reject*200
set app envs succeed
>>> set_app_envs replica.write_throttling_by_size 1000K*delay*100,2000K*reject*200
set app envs succeed
>>> set_app_envs replica.write_throttling_by_size 1000M*delay*100,2000M*reject*200
set app envs succeed
>>> get_app_envs
[app_envs]
replica.write_throttling          : 1000*delay*100,2000*reject*200
replica.write_throttling_by_size  : 1000M*delay*100,2000M*reject*200
```

## 表级读流控

从`v2.4.x`版本开始，Server端增加了基于读QPS的限流。读限流和写限流实现原理一样，使用方式类似。

**写流控设置示例：**

```shell
$ ./run.sh shell
>>> use temp
OK
>>> set_app_envs replica.read_throttling 1000*delay*100,2000*reject*200
set app envs succeed
>>> get_app_envs
[app_envs]
replica.read_throttling           : 1000*delay*100,2000*reject*200
```

