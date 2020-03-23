# Cross Cluster Replication

功能引入：
 - [6.5以beta feature引入](https://www.slideshare.net/elasticsearch/replicate-elasticsearch-data-with-crosscluster-replication-ccr)
 - [6.7.0+ 可用于生产](https://www.elastic.co/cn/blog/follow-the-leader-an-introduction-to-cross-cluster-replication-in-elasticsearch)

白金级功能，**收费**, 可以申请30天免费试用

本地集群版本必须大于等于远端集群版本，并且匹配

|Compatibility | 5.0→5.5 | 5.6 | 6.0→6.6 | 6.7 | 6.8 | 7.0 |7.1→7.x |
| ------       | ------- | ----| ------  | ----| ----| ----| ------|
|5.0→5.5       | yes     | yes | no      | no  | no |  no |  no    |
|5.6           | yes     | yes | yes     | yes | yes |  no |  no     |
|6.0→6.6       | no      | yes | yes     | yes | yes |  no |  no     |
|6.7           | no      | yes | yes     | yes | yes |  yes|  no     |
|6.8           | no      | yes | yes     | yes | yes |  yes | yes    |
|7.0           |no       | no  | no      | yes | yes |  yes | yes    |
|7.1→7.x       |no       | no  | no      | no  | yes |  yes | yes    |

## 1. 简介

ES提供了跨级群复制功能，其本质就是以一定的方式将数据在两个、多个集群间进行副本制作的过程。

其中被复制数据的集群称为“远端集群”，从“远端集群”复制数据的集群称为“本地集群”。

跨级群复制功能，主要有三种典型场景：

### 1.1 **灾难恢复 (DR) / 高可用性 (HA)**

- 当领导者索引不可用（如集群/数据中心中断）时，应用程序管理员或集群管理员必须为写入**显式选择**另一个索引，而这个索引很可能位于另一个集群中

### 1.2 **数据本地化**

通过复制，可以实现本地读，减少时延，降低成本

### 1.3 **集中式报告**

将数据从大量较小型集群复制回集中式报告集群。是数据本地化的一个特例。相当于有一个超级es集群，将所有关联小集群的信息都复制到超级集群中。
适用于跨大型网络进行查询的效率较低场景，最终数据会**在超级集群中进行本地分析和聚合**。


## 2. 原理机制

### 2.1 实现模式

跨级群功能是以索引级别进行管理的。围绕**leader-follower**模型设计

- 被复制的索引称为“leader index”，存在于远端集群上。
- “follower index”复制“leader index”数据，存在于本地集群上。

值得注意的是：
- 可以对leader index进行读写操作，但是只能对follower index进行读操作，不能进行写操作。也就是说follower index上的数据只能通过ccr从leader index上复制，然后由ccr写入
- 某个ccr过程中的follower index，可以是其他ccr过程中的leader index
- 可以通过[pause](https://www.elastic.co/guide/en/elasticsearch/reference/6.7/ccr-post-pause-follow.html)操作，实现停止从leader index上复制；然后通过[resume](https://www.elastic.co/guide/en/elasticsearch/reference/6.7/ccr-post-resume-follow.html)恢复复制
- follower index可以通过[unfollow](https://www.elastic.co/guide/en/elasticsearch/reference/6.7/ccr-post-unfollow.html)转换成正常的index，不过需要首先将follower index执行pause和[close](https://www.elastic.co/guide/en/elasticsearch/reference/6.7/indices-open-close.html)操作
- 不能将正常索引换成follower index
- 复制过程是由follower index主动去pull发起的，不会影响leader index索引文档

Es提供了两种方式配置，从而进行复制配置

- 通过[创建follower index操作](https://www.elastic.co/guide/en/elasticsearch/reference/6.7/ccr-put-follow.html)，指定需要复制的远端集群和leader index
- 通过[创建自动follow模式](https://www.elastic.co/guide/en/elasticsearch/reference/6.7/ccr-put-auto-follow-pattern.html)指定根据规则，当远端集群上有符合规则的leader index后，es自动创建follower index

### 2.2 复制机制

建立好复制配置以后，follower index创建好后，本地集群就会通过[remote recovery](https://www.elastic.co/guide/en/elasticsearch/reference/6.7/remote-recovery.html)将远端集群上的所有的lucene segments files拷贝到本地集群上来。

另外，当leader index上的mapping和settings发生变化时，follower index也会按需自动复制过来。但是并不是所有的settings修改都会被复制过来，比如修改leader index的副本数的变化，并不会被复制到follower index上来。

虽然ccr是基于索引进行配置管理，但是在具体复制过程中是以分片级别进行的。

1. follower index下面的每个分片会向leader index上的对应分片发送读取请求，拉取操作记录（可以是对应的主分片，也可能是副本分片响应这个读取请求）

2. leader index下面的分片收到follower分片发送过来的读取请求后，

- 如果有新操作发生，则会直接返回follower分片
  - 返回结果中操作数量受受创建follower index指定的`max_read_request_operation_count`和`max_read_request_size`配置控制
- 如果没有新操作的话，会判定是否超过了follower index配置的最大等待时间(`read_poll_timeout`，默认60s)
  - 如果超时，则返回一个没有新操作请求的响应给follower分片
  - 如果未超时，则等待，在等待过程中有新操作发生的话，会直接返回给follower分片

3. follower分片接收到响应后

- 一方面，更新一些统计信息，然后立马再次发送读取请求给leader分片（前提是write buffer未满）
- 于此同时，接收到的操作请求会被写入一个write buffer中，然后基于write buffer来创建bulk write requests来将操作写入到follower分片中
  - write buffer受创建follower index时指定的`max_write_buffer_count`， `max_write_buffer_size`控制
  - bulk write requests受`max_write_request_operation_count`，`max_write_request_size`参数控制
  - 如果write buffer满了，则不会向leader分片发送读取请求；只有等write buffer不再满了以后才会再次想leader分片发送读取请求

### 2.3 复制过程面临挑战

从2.2可以看出，只要follower能够拿到leader分片上所有操作记录的话，就能够保证所有的操作都能够体现到follower分片上。
但是由于es底层使用的lucene，而lucene本身出于优化搜索和节省空间的目的，过一段时间就会合并lucene segement files，而该合并操作会导致操作记录的丢失。

- 如果follower分片在lucene合并之前已经接收到被合并的操作，则一切顺利。
- 如果follower分片在lucene合并之前并没有接收到被合并的操作的话，follower分片就会遗漏操作，无法将所有操作历史记录复制到follower分片上。

#### 2.3.1 解决方案

严格来讲，es并没有完全解决上述挑战，但是为了尽量避免这种挑战带来的影响，es使用了[Lucene 中原生支持的“软删除”](https://issues.apache.org/jira/browse/LUCENE-8198)技术。

为了使用“软删除”技术，需要leader index在创建时指定`soft_deletes`支持（该配置不支持更改）。

使用了软删除支持以后，当leader分片上一个文档被更新或者删除时，这些操作会被保留一段时间，被称为历史记录保留租约，由创建leader index时指定的`index.soft_deletes.retention_lease.period`参数控制，默认12h.
当底层lucene想要执行合并操作时，会判断两个条件，只有任一条件满足才会执行合并操作：

- follower分片表示已经接收到需要合并的操作
- 需要合并的操作保留时间超过了历史记录保留租约时长

虽然引入了历史记录保留租约（默认为 12 个小时），确定了追随者在有可能远远落后并需要从领导者处重新引导之前可以离线的最长时间，但是有一个监控系统尽早的发现复制过程出现问题显得尤其重要。

### 2.4 异常处理

在复制过程中可能会发生错误，CCR会对发生的错误进行分类：

- **可恢复错误**，CCR即进入重试循环，一旦导致故障的情况得到解决，CCR 将立即继续复制。
- **致命错误**，无法恢复。但是有解决方案(重新创建follower index，但是这样会导致当前所有的lucene segment file丢失，重新初始化）。

```bash
POST /follower_index/_ccr/pause_follow

POST /follower_index/_close

PUT /follower_index/_ccr/follow?wait_for_active_shards=1
{
  "remote_cluster" : "remote_cluster",
  "leader_index" : "leader_index"
}
```

## 3. Demo

### 3.1 将集群和远端集群关联

在本地集群配置远端集群是进行CCR的基础，同时也是进行[Cross Cluster Search](https://www.elastic.co/guide/en/elasticsearch/reference/6.7/modules-cross-cluster-search.html)的基础。

```bash
# on the follower cluster
PUT /_cluster/settings
{
  "persistent" : {
    "cluster" : {
      "remote" : {
        "leader" : {
          "seeds" : [
            "127.0.0.1:9300"
          ]
        }
      }
    }
  }
}
```

至于seed个数可以随便写，但是默认的只会选择三个进行通信，可以通过参数`cluster.remote.connections_per_cluster`进行修改，如果想了解更详细的原理可以参考[Modules - Remote Clusters](https://www.elastic.co/guide/en/elasticsearch/reference/6.7/modules-remote-clusters.html#modules-remote-clusters)

集群间通信，使用的是TCP端口9300而不是9200的HTTP端口，因而不需要提供认证信息。

查看状态

```bash
GET /_remote/info
```

### 3.2 Manually follow the leader index

```bash
# on the leader cluster, create leader index
PUT /persons
{
  "settings" : {
    "index" : {
      "number_of_shards" : 1,
      "number_of_replicas" : 0,
      "soft_deletes" : {
        "enabled" : true
      }
    }
  },
  "mappings" : {
    "properties" : {
      "@timestamp" : {
        "type" : "date"
      },
      "age" : {
        "type" : "long"
      },
      "name" : {
        "type" : "keyword"
      }
    }
  }
}

# on the follower cluster, create follower index
PUT /persons/_ccr/follow?wait_for_active_shards=1
{
  "remote_cluster" : "leader",
  "leader_index" : "persons"
}
```

### 3.3 Auto follow the leader index

如果有很多个索引，然后每个索引都这样手动去follow的话，肯定易用性会大大折扣，因而es提供了根据指定规则自动follow。但是在此之前配置remote cluster是必须的

```bash
# on the follower cluster
PUT /_ccr/auto_follow/beats
{
  "remote_cluster" : "leader",
  "leader_index_patterns" :
  [
    "metricbeat-*",
    "packetbeat-*"
  ],
  "follow_index_pattern" : "{{leader_index}}-copy"
}
```

## 4. Other Sides

常用的ccr相关的接口
```
# Return all statistics related to CCR
GET /_ccr/stats

# Pause replication for a given index
POST /<follower_index>/_ccr/pause_follow
# Resume replication, in most cases after it has been paused
POST /<follower_index>/_ccr/resume_follow
{
}


# Convert follower index to normal index
## 1. Pause replication for a given index
POST /<follower_index>/_ccr/pause_follow
## 2. close the follower index
POST /<follower_index>/_close
## 3. Unfollow an index (stopping replication for the destination index), which first requires replication to be paused
POST /<follower_index>/_ccr/unfollow

# Statistics for a following index
GET /<index>/_ccr/stats

# Remove an auto-follow pattern
DELETE /_ccr/auto_follow/<auto_follow_pattern_name>
# View all auto-follow patterns, or get an auto-follow pattern by name
GET /_ccr/auto_follow/
GET /_ccr/auto_follow/<auto_follow_pattern_name>
```

## 5. Experiment

### 5.1 查看集群CCR信息



```bash
GET _ccr/stats
```

```json
{
  "auto_follow_stats" : {
    "number_of_failed_follow_indices" : 0,
    "number_of_failed_remote_cluster_state_requests" : 0,
    "number_of_successful_follow_indices" : 1,
    "recent_auto_follow_errors" : [ ],
    "auto_followed_clusters" : [
      {
        "cluster_name" : "elasticsearch2",
        "time_since_last_check_millis" : 55837,
        "last_seen_metadata_version" : 65
      }
    ]
  },
  "follow_stats" : {
    "indices" : [
      {
        "index" : "persons",
        "shards" : [
          {
            "remote_cluster" : "elasticsearch2",
            "leader_index" : "persons",
            "follower_index" : "persons",
            "shard_id" : 0,
            "leader_global_checkpoint" : 7,
            "leader_max_seq_no" : 7,
            "follower_global_checkpoint" : 7,
            "follower_max_seq_no" : 7,
            "last_requested_seq_no" : 7,
            "outstanding_read_requests" : 1,
            "outstanding_write_requests" : 0,
            "write_buffer_operation_count" : 0,
            "write_buffer_size_in_bytes" : 0,
            "follower_mapping_version" : 2,
            "follower_settings_version" : 6,
            "total_read_time_millis" : 197045,
            "total_read_remote_exec_time_millis" : 197034,
            "successful_read_requests" : 8,
            "failed_read_requests" : 0,
            "operations_read" : 8,
            "bytes_read" : 682,
            "total_write_time_millis" : 54,
            "successful_write_requests" : 8,
            "failed_write_requests" : 0,
            "operations_written" : 8,
            "read_exceptions" : [ ],
            "time_since_last_read_millis" : 12596
          }
        ]
      }
    ]
  }
}
```

对于单个索引的ccr信息的话，只有处于active状态的follower index才可以查看该接口，其他index请求的话，返回404错误

### 5.2 查看follower index信息

可以查看到配置信息和状态信息

```bash
GET persons/_ccr/info
```

```json
{
  "follower_indices" : [
    {
      "follower_index" : "persons",
      "remote_cluster" : "elasticsearch2",
      "leader_index" : "persons",
      "status" : "active",
      "parameters" : {
        "max_read_request_operation_count" : 5120,
        "max_write_request_operation_count" : 5120,
        "max_outstanding_read_requests" : 12,
        "max_outstanding_write_requests" : 9,
        "max_read_request_size" : "32mb",
        "max_write_request_size" : "9223372036854775807b",
        "max_write_buffer_count" : 2147483647,
        "max_write_buffer_size" : "512mb",
        "max_retry_delay" : "500ms",
        "read_poll_timeout" : "1m"
      }
    }
  ]
}
```

### follower index离线超时

follower index离线超时后，再次resume后，查看状态

```bash
GET persons-copy/_ccr/stats
```

```json
{
  "indices" : [
    {
      "index" : "persons-copy",
      "shards" : [
        {
          "remote_cluster" : "elasticsearch2",
          "leader_index" : "persons",
          "follower_index" : "persons-copy",
          "shard_id" : 0,
          "leader_global_checkpoint" : 6,
          "leader_max_seq_no" : 6,
          "follower_global_checkpoint" : 6,
          "follower_max_seq_no" : 6,
          "last_requested_seq_no" : 6,
          "outstanding_read_requests" : 1,
          "outstanding_write_requests" : 0,
          "write_buffer_operation_count" : 0,
          "write_buffer_size_in_bytes" : 0,
          "follower_mapping_version" : 2,
          "follower_settings_version" : 6,
          "total_read_time_millis" : 19,
          "total_read_remote_exec_time_millis" : 0,
          "successful_read_requests" : 0,
          "failed_read_requests" : 1,
          "operations_read" : 0,
          "bytes_read" : 0,
          "total_write_time_millis" : 0,
          "successful_write_requests" : 0,
          "failed_write_requests" : 0,
          "operations_written" : 0,
          "read_exceptions" : [
            {
              "from_seq_no" : 7,
              "retries" : 0,
              "exception" : {
                "type" : "resource_not_found_exception",
                "reason" : "Operations are no longer available for replicating. Maybe increase the retention setting [index.soft_deletes.retention.operations]?",
                "requested_operations_missing" : [
                  "7",
                  "9"
                ],
                "caused_by" : {
                  "type" : "illegal_state_exception",
                  "reason" : "Not all operations between from_seqno [7] and to_seqno [9] found; expected seqno [7]; found [Index{id='M1f1hXABTaraIKANr_61', type='_doc', seqNo=8, primaryTerm=1, version=1, autoGeneratedIdTimestamp=-1}]"
                }
              }
            }
          ],
          "time_since_last_read_millis" : 25481,
          "fatal_exception" : {
            "type" : "resource_not_found_exception",
            "reason" : "Operations are no longer available for replicating. Maybe increase the retention setting [index.soft_deletes.retention.operations]?",
            "requested_operations_missing" : [
              "7",
              "9"
            ],
            "caused_by" : {
              "type" : "illegal_state_exception",
              "reason" : "Not all operations between from_seqno [7] and to_seqno [9] found; expected seqno [7]; found [Index{id='M1f1hXABTaraIKANr_61', type='_doc', seqNo=8, primaryTerm=1, version=1, autoGeneratedIdTimestamp=-1}]"
            }
          }
        }
      ]
    }
  ]
}
```

这个时候其实就发生了致命错误错误，persons-copy中的文档永久的保存在了上次成功复制的版本上。即使在follower index执行resume之前几分钟内在leader index上的操作也不会体现到follower index上来。

要想让follower index继续有效的话，需要执行2.4中发生致命错误时可以恢复的操作。虽然是同名，但是其实follower index在lucene上的segments files会在执行这些操作时被删除，重新从leader index上复制过来。

```bash
POST /follower_index/_ccr/pause_follow

POST /follower_index/_close

PUT /follower_index/_ccr/follow?wait_for_active_shards=1
{
  "remote_cluster" : "remote_cluster",
  "leader_index" : "leader_index"
}
```

## 6. References

1. https://www.elastic.co/cn/blog/follow-the-leader-an-introduction-to-cross-cluster-replication-in-elasticsearch
2. https://www.elastic.co/cn/blog/cross-datacenter-replication-with-elasticsearch-cross-cluster-replication
3. https://www.elastic.co/guide/en/elasticsearch/reference/6.7/ccr-overview.html
4. https://www.elastic.co/guide/en/elasticsearch/reference/6.7/ccr-getting-started.html
5. https://www.elastic.co/guide/en/elasticsearch/reference/6.7/ccr-put-auto-follow-pattern.html
