---
permalink: administration/config
---

# 配置组成部分

Pegasus 的配置为 ini 格式，主要有以下组成部分：

* core：一个 Pegasus Service 内核引擎运行时的相关参数配置。
* network：RPC 组件的相关参数配置。
* 线程池相关：Pegasus 进程中启动的各个线程池的相关参数配置。
* app 相关：app 是 rDSN 中的一个概念，可以理解成分布式系统中的 “组件” 或者 “job”，例如 Pegasus 中的 MetaServer、ReplicaServer 就各是一个 app。一个进程内可以启动多个 app，针对每个 app，可以分别配置其行为，譬如名字、端口、线程池等。
* task 相关：task 也是 rDSN 中的一个概念，可以理解成 “异步任务”。比如一个 RPC 异步调用、一个异步文件 IO 操作、一个超时事件，都是一个 task。每种 task 都有定义一个唯一的名字。针对每种 task，都可以配置其行为，譬如 trace、profiler 等。
* 一致性协议相关：一致性 replication 协议的相关参数配置。
* RocksDB 相关：Pegasus 所使用的 RocksDB 的参数配置。
* 其他：Pegasus 中其他模块的参数配置，譬如日志、监控、Zookeeper 配置等。

配置文件中会涉及到一些 rDSN 的概念，对这些概念的进一步理解，请参见 [rDSN 项目](https://github.com/XiaoMi/rdsn)。

下面列举出了 Pegasus 配置文件的部分说明。这些配置项有些是和 client 通用的，比如 `app`、`task`、`threadpool` 等，其他是 server 端所独有的。要理解这些配置的真正含义，建议先阅读 [PacificA 论文](https://www.microsoft.com/en-us/research/publication/pacifica-replication-in-log-based-distributed-storage-systems/)，并了解清楚 rDSN 项目和 Pegasus 架构。

# 配置文件部分说明

```ini
;;;; 各个 app 配置项的默认模板
[apps..default]
run = true
count = 1

;;;; meta app 的配置项
[apps.meta]
type = meta
name = meta
arguments =
; meta 的运行端口
ports = 34601
; meta app 需要的线程池
pools = THREAD_POOL_DEFAULT,THREAD_POOL_META_SERVER,THREAD_POOL_META_STATE,THREAD_POOL_FD,THREAD_POOL_DLOCK,THREAD_POOL_FDS_SERVICE
run = true
; meta app 的实例个数，每个实例的运行端口依次为 ports, ports+1 等。可以用参数 "-app_list meta@<index>" 的方式启动指定的 app
count = 3

;;;; replica app 的配置项目
[apps.replica]
type = replica
name = replica
arguments =
ports = 34801
pools = THREAD_POOL_DEFAULT,THREAD_POOL_REPLICATION_LONG,THREAD_POOL_REPLICATION,THREAD_POOL_LOCAL_APP,THREAD_POOL_FD,THREAD_POOL_FDS_SERVICE,THREAD_POOL_COMPACT
run = true
count = 1

;;;; Pegasus 内核参数
[core]
; rDSN 相关概念，参见 rDSN 文档
tool = nativerun
; rDSN 相关概念，参见 rDSN 文档
toollets = profiler
; 启动时是否暂停以等待交互输入，通常用于调试目的
pause_on_start = false

; logging 级别
logging_start_level = LOG_LEVEL_INFO
; logging 的实现类
logging_factory_name = dsn::tools::simple_logger
; 进程退出时是否 flush 日志
logging_flush_on_exit = true

; 存放 data/log/coredump 等数据的默认目录
data_dir = %{app.dir}

;;;; 网络相关配置
[network]
; 负责网络 IO 的线程个数（含 timer 和 boost network）
io_service_worker_count = 4
; 每个 IP 到每个服务器的最大连接数，0 表示没有限制
conn_threshold_per_ip = 0

;;;; 线程池相关配置的默认模板
[threadpool..default]
; 线程池的线程数
worker_count = 4

;;;; 线程池 THREAD_POOL_REPLICATION 的配置
[threadpool.THREAD_POOL_REPLICATION]
; 线程池名称
name = replica
; 表示每个线程有一个自己的任务队列，且 task 会根据 hash 规则分派到特定的线程执行
partitioned = true
; 线程在 OS 中的调度优先级
worker_priority = THREAD_xPRIORITY_NORMAL
; 线程池中的线程数
worker_count = 24

;;;; 线程池 XXXXX 的相关配置
[threadpool.XXXXX]
.....

;;;; meta_server 的相关配置
[meta_server]
; MetaServer 的地址列表
server_list = %{meta.server.list}
; MetaServer 在元数据存储服务上的根目录，一个集群的不同 meta_server 要配成相同的值，不同的集群用不同的值
cluster_root = /pegasus/%{cluster.name}
; 元数据存储服务的实现类
meta_state_service_type = meta_state_service_zookeeper
; 元数据存储服务的初始化参数
meta_state_service_parameters =
; 分布式锁服务的实现类
distributed_lock_service_type = distributed_lock_service_zookeeper
; 分布式锁服务的初始化参数
distributed_lock_service_parameters = /pegasus/%{cluster.name}/lock
; 判断一个 ReplicaServer 是不是稳定运行的时间阈值
stable_rs_min_running_seconds = 600
; 一个 ReplicaServer 最多可以失联的次数。如果 ReplicaServer 失联数大于该值，MetaServer 则会将其加入黑名单，避免频繁重连带来的不稳定问题
max_succssive_unstable_restart = 5
; 负载均衡器的实现类
server_load_balancer_type = greedy_load_balancer
; 当一个 secondary 被移除后，等待它加回集群的最长时间阈值。如果超过这个时间阈值 secondary 还未恢复，将会尝试在其他节点上补充新的 secondary。， 
replica_assign_delay_ms_for_dropouts = 300000
; 如果可用节点比例小于该阈值，MetaServer 会进入 freezed 的保护状态
node_live_percentage_threshold_for_update = 50
; 如果可用的节点个数低于该阈值，MetaServer 也会进入 freezed 的保护状态
min_live_node_count_for_unfreeze = 3
; 表删除后税局的默认保留时间
hold_seconds_for_dropped_app = 604800
; MetaServer 启动时的默认 function_level 状态。steady 表示不进行负载均衡的稳定状态
meta_function_level_on_start = steady
; 当元数据存储中没有表的数据时，是否从 replica server 恢复表
recover_from_replica_server = false
; 一个 replica group 中最多保留的副本数 (可用副本 + 尸体)
max_replicas_in_group = 4

;;;; 集群在启动时要创建的表
[replication.app]
; 表名
app_name = temp
; 存储引擎类型，"pegasus" 表示基于 rocksdb 实现的存储引擎。当前只可选 pegasus
app_type = pegasus
; 分片数
partition_count = 8
; 每个分片的副本个数
max_replica_count = 3
; rDSN 参数，对于 app_type = pegasus 需要设置为 true
stateful = true
package_id =

;;;; 一致性协议相关配置，很多概念和 PacificA 相关
[replication]
; shared log 的目录。自 Pegasus 2.6.0 开始已弃用，但如果从旧版本升级，请保留该值，并且不要修改该值。
slog_dir = /home/work/ssd1/pegasus/@cluster@
; replica 数据存储的路径列表，建议一块磁盘配置一个项。tag 为磁盘的标记名
data_dirs = tag1:/home/work/ssd2/pegasus/@cluster@,tag2:/home/work/ssd3/pegasus/@cluster@
; 黑名单文件，文件中每行是一个需忽略掉的路径，主要用于过滤坏盘
data_dirs_black_list_file = /home/work/.pegasus_data_dirs_black_list
; ReplicaServer 启动时是否要拒绝掉客户端读写请求
deny_client_on_start = false
; ReplicaServer 启动时是否等待一段时间后才开始连接 MetaServer 来触发 failure detector 超时
delay_for_fd_timeout_on_start = false
; 是否禁用 primary 定期生成一个空的写操作以检查 group 状态的功能
empty_write_disabled = false
; primary 给 secondaries 发 prepare 的两阶段提交超时时间
prepare_timeout_ms_for_secondaries = 1000
; primary 给 learner 发 prepare 的超时时间
prepare_timeout_ms_for_potential_secondaries = 3000
; 是否禁止掉客户端写请求的 batch 功能
batch_write_disabled = false
; 保留多少个已经 commit 的写请求在队列中
staleness_for_commit = 10
; prepare_list 的容量
max_mutation_count_in_prepare_list = 110
; 想要成功进行一次写操作，最少需要多少个副本
mutation_2pc_min_replica_count = 2
; 是否禁用 primary 定期推送 group-check 请求给其他成员的功能
group_check_disabled = false
; primary 发送 group-check 的时间间隔
group_check_interval_ms = 10000
; 是否禁用定期 checkpoint 的生成
checkpoint_disabled = false
; checkpoint 的尝试触发时间间隔。注意尝试触发并不一定生成 checkpoint
checkpoint_interval_seconds = 100
; checkpoint 的强制触发时间间隔，强制触发会将 memtable 的数据刷出
checkpoint_max_interval_hours = 2
; 是否禁用 replica 的统计信息。名字中的 "gc" 是历史遗留问题.
gc_disabled = false
; 做 replica 统计信息的间隔时间。名字中的 "gc" 是历史遗留问题.
gc_interval_ms = 30000
; 如果一个 replica 需要关闭，在内存中保留多长时间
gc_memory_replica_interval_ms = 600000
; 一个因为 IO 错误关闭掉的 replica，在磁盘保留多长时间
gc_disk_error_replica_interval_seconds = 86400
; 是否关闭 failure detector
fd_disabled = false
; failure detector 检查远程副本健康状况的间隔秒数
fd_check_interval_seconds = 2
; failure detector 向远程副本发送 beacon 的间隔秒数
fd_beacon_interval_seconds = 3
fd_lease_seconds = 9
fd_grace_seconds = 10
; 每一个 private log 多大，超过了该阈值就滚动到下一个文件
log_private_file_size_mb = 32
; 要保留的无用的 private log 最大大小。注意：只有当 log_private_reserve_max_size_mb 和 log_pripate_reserve_max_time_seconds 都满足时，才能保留无用的日志。
log_private_reserve_max_size_mb = 1000
; 要保留的无用的 private log 最长时间。注意：只有当 log_private_reserve_max_size_mb 和 log_pripate_reserve_max_time_seconds 都满足时，才能保留无用的日志。
log_private_reserve_max_time_seconds = 3600
; shared log 相关配置（2.6 开始已废弃）
log_shared_file_size_mb = 128
log_shared_file_count_limit = 64
log_shared_batch_buffer_kb = 0
log_shared_force_flush = false
log_shared_pending_size_throttling_threshold_kb = 0
log_shared_pending_size_throttling_delay_ms = 0
; 是否禁用 replica server 向 meta server 定期同步本机所服务的 replica 功能。
config_sync_disabled = false
; replica server 向 meta server 同步本机所服务的 replica 的时间间隔。
config_sync_interval_ms = 30000
; meta server 跑负载均衡的周期
lb_interval_ms = 10000

;;;; pegasus server 层相关配置
[pegasus.server]
; 是否打印 Pegasus 中反应 RocksDB 运行情况的调试日志
rocksdb_verbose_log = false
; 如果 Get/Multi-Get 操作的耗时大于此配置，则会打印一条 warning 日志。
rocksdb_slow_query_threshold_ns = 1000000
; 如果 Get 操作的 key-value 大小大于此配置，则会打印一条 warning 日志，0表示从不打印。
rocksdb_abnormal_get_size_threshold = 1000000
; 如果 Multi-Get 操作的 key-value 大小之和大于此配置，则会打印一条 warning 日志，0表示从不打印。
rocksdb_abnormal_multi_get_size_threshold = 10000000
; 如果 Multi-Get 操作的 scan 迭代数大于此配置，则会打印一条 warning 日志，0表示从不打印。
rocksdb_abnormal_multi_get_iterate_count_threshold = 1000

;;; RocksDB 相关配置，请查阅 https://github.com/facebook/rocksdb/blob/main/include/rocksdb/{options.h, advanced_options.h}
; 对应 RocksDB 的 options.write_buffer_size
rocksdb_write_buffer_size = 67108864
; 对应 RocksDB 的 options.max_write_buffer_number
rocksdb_max_write_buffer_number = 3
; 对应 RocksDB 的 options.max_background_flushes，同一个进程中的所有 RocksDB 实例共享这些 flush 线程。
rocksdb_max_background_flushes = 4
; 对应 RocksDB 的 options.max_background_compactions，同一个进程中的所有 RocksDB 实例共享这些 compaction 线程。
rocksdb_max_background_compactions = 12
; 对应 RocksDB 的 options.num_levels
rocksdb_num_levels = 6
; 对应 RocksDB 的 options.target_file_size_base
rocksdb_target_file_size_base = 67108864
; 对应 RocksDB 的 options.target_file_size_multiplier
rocksdb_target_file_size_multiplier = 1
; 对应 RocksDB 的 options.max_bytes_for_level_base
rocksdb_max_bytes_for_level_base = 671088640
; 对应 RocksDB 的 options.rocksdb_max_bytes_for_level_multiplier
rocksdb_max_bytes_for_level_multiplier = 10
; 对应 RocksDB 的 options.level0_file_num_compaction_trigger
rocksdb_level0_file_num_compaction_trigger = 4
; 对应 RocksDB 的 options.level0_slowdown_writes_trigger
rocksdb_level0_slowdown_writes_trigger = 30
; 对应 RocksDB 的 options.level0_stop_writes_trigger
rocksdb_level0_stop_writes_trigger = 60
; 对应于 RocksDB 的 options.compression。可用配置：'[none|snappy|zstd|lz4]' 适用于所有 level 1 及更高 level，以及对 0,1,... 每个 level 的 'per_level:[none|snappy|zstd|lz4],[none|snappy|zstd|lz4],...'，最后一种压缩类型将用于配置列表中未指定的级别。
rocksdb_compression_type = lz4
; 是否禁用 RocksDB 的 block cache
rocksdb_disable_table_block_cache = false
; 进程中所有 RocksDB 实例共享的 Block Cache 容量，以 bytes 为单位
rocksdb_block_cache_capacity = 10737418240
; Block cache 的 shard 位数，block cache 被分为 2^n 个 shard，以减少锁竞争。-1 表示自动确定
rocksdb_block_cache_num_shard_bits = -1
; 是否禁用 RocksDB 布隆过滤器
rocksdb_disable_bloom_filter = false

;;; 监控相关配置，部分与 Open-Falcon 相关。自 2.6 开始已被废弃。
perf_counter_update_interval_seconds = 10
perf_counter_enable_logging = false
falcon_host = 127.0.0.1
falcon_port = 1988
falcon_path = /v1/push

;;;; 监控实现类相关
[components.pegasus_perf_counter_number_percentile_atomic]
counter_computation_interval_seconds = 10

;;;; task 相关默认配置
[task..default]
is_trace = false
is_profile = false
allow_inline = false
rpc_call_header_format = NET_HDR_DSN
rpc_call_channel = RPC_CHANNEL_TCP
rpc_timeout_milliseconds = 5000	
disk_write_fail_ratio = 0.0
disk_read_fail_ratio = 0.0

;;;; task RPC_L2_CLIENT_READ 的相关配置，未指定项继承自 'default' 模板
[task.RPC_L2_CLIENT_READ]
is_profile = true
profiler::inqueue = false
profiler::queue = false
profiler::exec = false
profiler::qps = false
profiler::cancelled = false
profiler::latency.server = false

;;;; Zookeeper 相关配置
[zookeeper]
hosts_list = 127.0.0.1:22181
timeout_ms = 10000
logfile = zoo.log

;;;; simple_logger 相关配置
[tools.simple_logger]
short_header = true
fast_flush = false
max_number_of_log_files_on_disk = 20
stderr_start_level = LOG_LEVEL_WARNING
```

# 配置建议

* 配置文件中所有需要使用服务器地址的配置项，都建议使用 IP 地址。
* 大部分配置项，建议使用默认值。
* 在理解配置项的作用和影响的前提下，可以根据需要更改配置值。
* 对于配置项的进一步了解，可以查看源代码。
