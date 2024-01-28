---
permalink: administration/config
---

# Configuration components

The configuration of Pegasus is in ini format, which mainly consists of the following components:

* core: The configurations are related to the runtime of a Pegasus Service kernel engine.
* network: The configurations are related to the RPC component.
* Thread pools: The configurations are for each thread pool in the Pegasus Service.
* app：`app` is a concept in rDSN that can be understood as a _component_ or _job_ in a distributed system. For example, MetaServer and ReplicaServer in Pegasus are each apps. Multiple apps can be launched within a process, and for each app, its behavior can be configured separately, such as name, port, thread pool, etc.
* task: Task is also a concept in rDSN, which can be understood as _asynchronous task_. For example, an RPC asynchronous call, an asynchronous file IO operation, or a timer event are all tasks. Each task has a unique name defined. For each task, its behavior can be configured, such as trace, profiler, etc.
* Consistency protocol: Configurations are related to consistency replication protocol.
* RocksDB: The configurations of RocksDB that Pegasus uses.
* Others: The configurations of other modules in Pegasus, such as logging, monitoring, Zookeeper configuration, etc.

The configuration file involves some rDSN concepts. For further understanding of these concepts, please refer to the [rDSN project](https://github.com/XiaoMi/rdsn).

Below is a partial explanation of the Pegasus configuration file. Some of these configuration items are common to clients, such as `app`, `task`, `threadpool`, etc., while others are unique to the server side. To understand the true meaning of these configurations, it is recommended to first read the [PacificA paper](https://www.microsoft.com/en-us/research/publication/pacifica-replication-in-log-based-distributed-storage-systems/) and have a clear understanding of the rDSN project and Pegasus architecture.

# Detail description of configuration file

```ini
;;;; Default templates for various 'app'.
[apps..default]
run = true
count = 1

;;;; Configurations for 'meta app'.
[apps.meta]
type = meta
name = meta
arguments =
; Listening port of 'meta app'.
ports = 34601
; Thread pool required of 'meta app'.
pools = THREAD_POOL_DEFAULT,THREAD_POOL_META_SERVER,THREAD_POOL_META_STATE,THREAD_POOL_FD,THREAD_POOL_DLOCK,THREAD_POOL_FDS_SERVICE
run = true
; The number of instances of the 'meta app', with each instance running on ports, ports+1, etc. The parameters '-app_list meta@<index>' can be used to launch the specified app.
count = 3

;;;; Configurations for 'replica app'.
[apps.replica]
type = replica
name = replica
arguments =
ports = 34801
pools = THREAD_POOL_DEFAULT,THREAD_POOL_REPLICATION_LONG,THREAD_POOL_REPLICATION,THREAD_POOL_LOCAL_APP,THREAD_POOL_FD,THREAD_POOL_FDS_SERVICE,THREAD_POOL_COMPACT
run = true
count = 1

;;;; Kernel configurations of Pegasus service.
[core]
; rDSN related concepts, can be found in the rDSN documentation.
tool = nativerun
; rDSN related concepts, can be found in the rDSN documentation.
toollets = profiler
; Whether to pause during startup to wait for interactive input, often for debugging perpose.
pause_on_start = false

; Logs with level larger than or equal to this level be logged.
logging_start_level = LOG_LEVEL_INFO
; The implementation class of logging.
logging_factory_name = dsn::tools::simple_logger
; Whether to flush the logs when the process exits.
logging_flush_on_exit = true

; The default directory to place the all the data/log/coredump, etc..
data_dir = %{app.dir}

;;;; Network related configurations.
[network]
; The thread number of IO service (timer and boost network).
io_service_worker_count = 4
; The maximum connection count to each server per ip, 0 means no limit.
conn_threshold_per_ip = 0

;;;; The default template configurations for thread pool.
[threadpool..default]
; The number of threads in the thread pool.
worker_count = 4

;;;; The configurations of THREAD_POOL_REPLICATION thead pool.
[threadpool.THREAD_POOL_REPLICATION]
; Thread pool name.
name = replica
; Whether each thread has its own task queue, and tasks are assigned to a specific thread for execution based on a hash rule to reduce lock contention. Otherwise, the threads share a single queue.
partitioned = true
; The scheduling priority of threads in OS.
worker_priority = THREAD_xPRIORITY_NORMAL
; The number of threads in the thread pool.
worker_count = 24

;;;; The configurations of XXXXX thead pool.
[threadpool.XXXXX]
.....

;;;; The configurations of 'meta_server'.
[meta_server]
; Address list for MetaServer.
server_list = %{meta.server.list}
; The root of the cluster meta state service to be stored on remote storage. Different meta servers in the same cluster need to be configured with the same value, while different clusters using different values if they share the same remote storage.
cluster_root = /pegasus/%{cluster.name}
; The implementation class of metadata storage service.
meta_state_service_type = meta_state_service_zookeeper
; Initialization parameters for metadata storage services.
meta_state_service_parameters =
; The implementation class of distributed lock service.
distributed_lock_service_type = distributed_lock_service_zookeeper
; Initialization parameters for distributed lock services.
distributed_lock_service_parameters = /pegasus/%{cluster.name}/lock
; The time threshold for determining whether a ReplicaServer is running stably.
stable_rs_min_running_seconds = 600
; The maximum number of times a ReplicaServer can be disconnected. If the number of a ReplicaServer disconnects exceeds this threshold, MetaServer will add it to the blacklist to avoid instability caused by frequent reconnection.
max_succssive_unstable_restart = 5
; The implementation class of load balancer.
server_load_balancer_type = greedy_load_balancer
; The maximum time threshold for waiting for a secondary replica to re-join the cluster after it has been removed. If the secondary replica has not yet recovered beyond this time threshold, an attempt will be made to add new secondary replicas on other nodes.
replica_assign_delay_ms_for_dropouts = 300000
; If the proportion of ALIVE nodes is less than this threshold, MetaServer will enter the 'freezed' protection state.
node_live_percentage_threshold_for_update = 50
; If the number of ALIVE nodes is less than this threshold, MetaServer will also enter the 'freezed' protection state.
min_live_node_count_for_unfreeze = 3
; Default time in seconds to reserve the data of deleted tables.
hold_seconds_for_dropped_app = 604800
; The default function_level state when MetaServer starts. The 'steady' represents a stable state without load balancing.
meta_function_level_on_start = steady
; Whether to recover tables from replica servers when there is no data of the tables in remote storage
recover_from_replica_server = false
; The maximum number of replicas retained in a replica group (alive + dead replicas).
max_replicas_in_group = 4

;;;; The tables to be created during cluster startup.
[replication.app]
; Table name.
app_name = temp
; The storage engine type, 'pegasus' represents the storage engine based on Rocksdb. Currently, only 'pegasus' is available.
app_type = pegasus
; Partition count.
partition_count = 8
; The maximum replica count of each partition.
max_replica_count = 3
; Whether this is a stateful table, it must be true if 'app_type = pegasus'.
stateful = true
package_id =

;;;; Consistency protocol related configurations, many concepts are related to PacificA.
[replication]
; The shared log directory. Deprecated since Pegasus 2.6.0, but leave it and do not modify the value if upgrading from older versions.
slog_dir = /home/work/ssd1/pegasus/@cluster@
; A list of directories for replica data storage, it is recommended to configure one item per disk. 'tag' is the tag name of the directory.
data_dirs = tag1:/home/work/ssd2/pegasus/@cluster@,tag2:/home/work/ssd3/pegasus/@cluster@
; Blacklist file, where each line is a path that needs to be ignored, mainly used to filter out bad drives.
data_dirs_black_list_file = /home/work/.pegasus_data_dirs_black_list
; Whether to deny client read and write requests when starting the server.
deny_client_on_start = false
; Whether to wait some time to start connecting to MetaServer when starting ReplicaServer to make failure detector timeout.
delay_for_fd_timeout_on_start = false
; Whether to disable the function of primary replicas periodically generating empty write operations to check the group status.
empty_write_disabled = false
; The timeout in millisecond for the primary replicas to send prepare requests to the secondaries in two phase commit.
prepare_timeout_ms_for_secondaries = 1000
; The timeout in millisecond for the primary replicas to send prepare requests to the learners in two phase commit.
prepare_timeout_ms_for_potential_secondaries = 3000
; Whether to disable auto-batch of replicated write requests.
batch_write_disabled = false
; The maximum number of two-phase commit rounds are allowed.
staleness_for_commit = 10
; The maximum number of mutations allowed in prepare list.
max_mutation_count_in_prepare_list = 110
; The minimum number of ALIVE replicas under which write is allowed. It's valid if larger than 0, otherwise, the final value is based on 'app_max_replica_count'.
mutation_2pc_min_replica_count = 2
; Whether to disable the primary replicas to send group-check requests to other replicas periodically. 
group_check_disabled = false
; The interval in milliseconds for the primary replicas to send group-check requests.
group_check_interval_ms = 10000
; Whether to disable to generate replica checkpoints periodically.
checkpoint_disabled = false
; The interval in seconds to generate replica checkpoints. Note that the checkpoint may not be generated when attempt.
checkpoint_interval_seconds = 100
; The maximum time interval in hours of replica checkpoints must be generated.
checkpoint_max_interval_hours = 2
; Whether to disable replica statistics. The name contains 'gc' is for legacy reason.
gc_disabled = false
; The interval milliseconds to do replica statistics. The name contains 'gc' is for legacy reason.
gc_interval_ms = 30000
; The milliseconds of a replica remain in memory for quick recover aim after it's closed in healthy state (due to LB).
gc_memory_replica_interval_ms = 600000
; The interval milliseconds to GC error replicas, which are in directories suffixed with '.err'.
gc_disk_error_replica_interval_seconds = 86400
; Whether to disable failure detection.
fd_disabled = false
; The interval seconds of failure detector to check healthness of remote peers
fd_check_interval_seconds = 2
; The interval seconds of failure detector to send beacon message to remote peers
fd_beacon_interval_seconds = 3
; The lease in seconds get from remote FD master
fd_lease_seconds = 9
; The grace in seconds assigned to remote FD slaves
fd_grace_seconds = 10
; The maximum size (MB) of private log segment file.
log_private_file_size_mb = 32
; The maximum size of useless private log to be reserved. NOTE: only when 'log_private_reserve_max_size_mb' and 'log_private_reserve_max_time_seconds' are both satisfied, the useless logs can be reserved.
log_private_reserve_max_size_mb = 1000
; The maximum time in seconds of useless private log to be reserved. NOTE: only when 'log_private_reserve_max_size_mb' and 'log_private_reserve_max_time_seconds' are both satisfied, the useless logs can be reserved.
log_private_reserve_max_time_seconds = 3600
; Shared log related configuration (removed since 2.6).
log_shared_file_size_mb = 128
log_shared_file_count_limit = 64
log_shared_batch_buffer_kb = 0
log_shared_force_flush = false
log_shared_pending_size_throttling_threshold_kb = 0
log_shared_pending_size_throttling_delay_ms = 0
; Whether to disable sending replica config-sync requests periodically to meta server.
config_sync_disabled = false
; The interval milliseconds of replica server to send replica config-sync requests to meta server.
config_sync_interval_ms = 30000
; The interval milliseconds of meta server to execute load balance.
lb_interval_ms = 10000

;;;; Pegasus Server Layer related configurations
[pegasus.server]
; Whether to print RocksDB related verbose log for debugging
rocksdb_verbose_log = false
; A warning log will be print if the duration of Get/Multi-Get operation is larger than this config.
rocksdb_slow_query_threshold_ns = 1000000
; A warning log will be print if the key-value size of Get operation is larger than this config, 0 means never print.
rocksdb_abnormal_get_size_threshold = 1000000
; A warning log will be print if the total key-value size of Multi-Get operation is larger than this config, 0 means never print.
rocksdb_abnormal_multi_get_size_threshold = 10000000
; A warning log will be print if the scan iteration count of Multi-Get operation is larger than this config, 0 means never print.
rocksdb_abnormal_multi_get_iterate_count_threshold = 1000

;;; RocksDB related configurations, see details https://github.com/facebook/rocksdb/blob/main/include/rocksdb/{options.h, advanced_options.h} 
; Corresponding to RocksDB's options.write_buffer_size
rocksdb_write_buffer_size = 67108864
; Corresponding to RocksDB's options.max_write_buffer_number
rocksdb_max_write_buffer_number = 3
; Corresponding to RocksDB's options.max_background_flushes, the flush threads are shared among all RocksDB's instances in the process.
rocksdb_max_background_flushes = 4
; Corresponding to RocksDB's options.max_background_compactions, the compaction threads are shared among all RocksDB's instances in the process.
rocksdb_max_background_compactions = 12
; Corresponding to RocksDB's options.num_levels
rocksdb_num_levels = 6
; Corresponding to RocksDB's options.target_file_size_base
rocksdb_target_file_size_base = 67108864
; Corresponding to RocksDB's options.target_file_size_multiplier
rocksdb_target_file_size_multiplier = 1
; Corresponding to RocksDB's options.max_bytes_for_level_base
rocksdb_max_bytes_for_level_base = 671088640
; Corresponding to RocksDB's options.rocksdb_max_bytes_for_level_multiplier
rocksdb_max_bytes_for_level_multiplier = 10
; Corresponding to RocksDB's options.level0_file_num_compaction_trigger
rocksdb_level0_file_num_compaction_trigger = 4
; Corresponding to RocksDB's options.level0_slowdown_writes_trigger
rocksdb_level0_slowdown_writes_trigger = 30
; Corresponding to RocksDB's options.level0_stop_writes_trigger
rocksdb_level0_stop_writes_trigger = 60
; Corresponding to RocksDB's options.compression. Available config: '[none|snappy|zstd|lz4]' for all level 1 and higher levels, and 'per_level:[none|snappy|zstd|lz4],[none|snappy|zstd|lz4],...' for each level 0,1,..., the last compression type will be used for levels not specified in the list.
rocksdb_compression_type = lz4
; Whether to disable RocksDB's block cache
rocksdb_disable_table_block_cache = false
; The Block Cache capacity shared by all RocksDB instances in the process, in bytes
rocksdb_block_cache_capacity = 10737418240
; The number of shard bits of the block cache, it means the block cache is sharded into 2^n shards to reduce lock contention. -1 means automatically determined
rocksdb_block_cache_num_shard_bits = -1
; Whether to disable RocksDB bloom filter
rocksdb_disable_bloom_filter = false

;;; Perf counter related configurations, some of which are related to the Open-Falcon. Deprecated since Pegasus 2.6.0
perf_counter_update_interval_seconds = 10
perf_counter_enable_logging = false
falcon_host = 127.0.0.1
falcon_port = 1988
falcon_path = /v1/push

;;;; The implementation of some perf counters
[components.pegasus_perf_counter_number_percentile_atomic]
counter_computation_interval_seconds = 10

;;;; task related default configurations
[task..default]
is_trace = false
is_profile = false
allow_inline = false
rpc_call_header_format = NET_HDR_DSN
rpc_call_channel = RPC_CHANNEL_TCP
rpc_timeout_milliseconds = 5000	
disk_write_fail_ratio = 0.0
disk_read_fail_ratio = 0.0

;;;; task RPC_L2_CLIENT_READ related configurations, unspecified configs are inherited from the 'default' template
[task.RPC_L2_CLIENT_READ]
is_profile = true
profiler::inqueue = false
profiler::queue = false
profiler::exec = false
profiler::qps = false
profiler::cancelled = false
profiler::latency.server = false

;;;; Zookeeper related configurations
[zookeeper]
hosts_list = 127.0.0.1:22181
timeout_ms = 10000
logfile = zoo.log

;;;; simple_logger related configurations
[tools.simple_logger]
short_header = true
fast_flush = false
max_number_of_log_files_on_disk = 20
stderr_start_level = LOG_LEVEL_WARNING
```

# Configuration suggestions

* It is recommended to use IP addresses for all items in the configuration file that require the use of server addresses.
* It is recommended to use default values for most configuration items.
* You can change the configuration values as needed only if you understand the meaning and impact of configuration items.
* For further understanding of configuration items, please refer to the source code.
