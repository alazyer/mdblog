# Cross Cluster Replication

版本要求：6.7.0+

白金级功能，**收费**

##  ## 使用场景

1. **灾难恢复 (DR) / 高可用性 (HA)**：当主集群挂了以后，备用集群可以立马顶上
2. **数据本地化**：通过复制，可以实现本地读，减少时延，降低成本
3. **集中式报告**：将数据从大量较小型集群复制回集中式报告集群。是数据本地化的一个特例。相当于有一个超级es集群，将所有关联小集群的信息都复制到超级集群中。适用于跨大型网络进行查询的效率较低场景，通过有复制**在超级集群中进行本地分析和聚合**。

## Example

### 将集群和远程集群关联

```bash
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

查看状态

```bash
GET /_remote/info
```

### Manually create leader/follower index

```bash
# create leader index
PUT /server-metrics
{
  "settings" : {
    "index" : {
      "number_of_shards" : 1,
      "number_of_replicas" : 0
    }
  },
  "mappings" : {
    "properties" : {
      "@timestamp" : {
        "type" : "date"
      },
      "accept" : {
        "type" : "long"
      },
      "deny" : {
        "type" : "long"
      },
      "host" : {
        "type" : "keyword"
      },
      "response" : {
        "type" : "float"
      },
      "service" : {
        "type" : "keyword"
      },
      "total" : {
        "type" : "long"
      }
    }
  }
}

# create follower index
PUT /server-metrics-copy/_ccr/follow?wait_for_active_shards=1
{
  "remote_cluster" : "leader",
  "leader_index" : "server-metrics"
}
```

### Auto create leader/follower index

```bash
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



## References

1. https://www.elastic.co/cn/blog/follow-the-leader-an-introduction-to-cross-cluster-replication-in-elasticsearch
2. https://www.elastic.co/guide/en/elastic-stack-overview/current/ccr-getting-started.html
3. https://www.elastic.co/guide/en/elasticsearch/reference/7.3/ccr-put-auto-follow-pattern.html