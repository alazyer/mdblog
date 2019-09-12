# 通过ILM实现ES冷热备份

[TOC]

## Intro

> You control how indices are handled as they age by **attaching a lifecycle policy to the index template** used to create them. You can update the policy to **modify the lifecycle of both new and existing indices**.
>
> For time series indices, there are four stages in the index lifecycle:
>
> - Hot—the index is actively being updated and queried.
> - Warm—the index is no longer being updated, but is still being queried.
> - Cold—the index is no longer being updated and is seldom queried. The information still needs to be searchable, but it’s okay if those queries are slower.
> - Delete—the index is no longer needed and can safely be deleted.

通过给index template关联一个index声明周期管理策略来控制index。更新策略的话，会同时影响新创建和已有的index。

对于时间序列形式的index，在index声明周期中有4个阶段（stages）：

- hot：此阶段index会被活跃的更新和查询
- warm：此阶段index不会被更新，但是仍然会被查询
- cold：此阶段index不会被更新，而且很少被查询。因而index中保存的信息只需要能被被搜索到就行，性能慢点无所谓
- delete：此阶段index已经不再需要了，可以被删除

## 设置节点属性

```bash
node.attr.data=hot
```

由于我们的es是通过docker容器起的es，所以在启动时添加上述环境变量。为es节点添加data=hot属性，属性名称和属性值任意。这个只要后面在创建策略中使用的 属性名和属性值 对应起来就行。

## 创建一个ILM策略

```bash
PUT _ilm/policy/datastream_policy   
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "set_priority": {
            "priority": 50
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "forcemerge": {
            "max_num_segments": 1
          },
          "shrink": {
            "number_of_shards": 1
          },
          "allocate": {
            "require": {
              "data": "warm"
            }
          },
          "set_priority": {
            "priority": 25
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "set_priority": {
            "priority": 0
          },
          "allocate": {
            "require": {
              "data": "cold"
            }
          }
        }
      },
      "delete": {
        "min_age": "1000d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

这个策略当index创建超过7天后会被转存到warm节点，转存到warm后30天后会被转存到cold节点，之后1000天会被删除

- 索引进入不同phrase的依据是通过min_age控制的。默认min_age为0s
- ilm会确保上一个phrase是complete才会计算下一个的min_age

## 将ILM策略关联到index模板

```bash
PUT _template/datastream_template
{
  "index_patterns": ["datastream-*"],                 # 关联索引规则         
  "settings": {
    "number_of_shards": 1,                            # 分片数
    "number_of_replicas": 0,                          # 副本数
    "index.lifecycle.name": "datastream_policy",      # 上一步创建的策略名称
    "index.routing.allocation.require.data": "hot"    # 创建index时，首先放到hot节点上
  }
}
```

- 当policy中定义了rollover的action是，在创建模板的时候需要制定`index.lifecycle.rollover_alias`
- 于此同时，创建index时，需要指定aliases

## 创建index

```bash
PUT datastream-000001
{}
```

为了方便，我们创建一个空的索引，并未指定mapping等属性。但是index的名字符合上述datastream_template中index_patterns中定义规则。

## 检查index状态

```
GET datastream-000001/_ilm/explain
```

## Phase Execution

> The current phase definition, of an index’s policy being executed, is stored in the **index’s metadata**. The **phase and its actions** are compiled into a series of **discrete steps** that are executed sequentially.

index当前所处的phase被放到了index的metadata中进行保存

每个phase可以定义若干actions，然后这些actions会被编译成一系列的steps，然后依次执行。用户在policy中定义了一系列actions，具体执行顺序是由ILM决定，不依赖policy中指定顺序。

## 常用Actions

### rollover

只适用于hot。

- 需要保证index名字以数字结尾
- 被管理的index需要设置`index.lifecycle.rollover_alias`

当指定的index满足指定条件后，会新创建index

```console
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover" : {
            "max_size": "100GB",
            "max_docs": 10000,
            "max_age": "1d"
          }
        }
      }
    }
  }
}
```

### allocate

适用于warm和cold。用于将index下的shards迁移到指定的node上

```console
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "warm": {
        "actions": {
          "allocate" : {
            "number_of_replicas" : 2,
            "include": {},
            "exclude": {},
            "require": 
          }
        }
      }
    }
  }
}
```

### set_priority

适用于hot，warm和cold。用于为index指定priority

具有高priority的index会在node重启后，优先被恢复。

一般策略是hot > warm > cold

如果没有设置本action的话，默认priority为1

### unfollow

适用于hot，warm和cold。并且这个action不需要任何参数

可以显示调用这个action，于此同时，执行rollover和shrink也会隐式的调用这个action

用于将ccr follower index改成一个regular index？

### readyonly

只适用用warm。并且这个action不需要任何参数

用户在warm phrase将index变成只读的

### forcemerge

只适用用warm。会在执行前，调用readonly

调用force merges将index merge成为指定个数的segments

```console
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "warm": {
        "actions": {
          "forcemerge" : {
            "max_num_segments": 1
          }
        }
      }
    }
  }
}
```

### shrink

只适用用warm。会在执行前，调用readonly

调用shrink 根据老的index创建一个新的index。但是primary shards数量会变成指定的数量

```console
PUT _ilm/policy/my_policy
{
  "policy": {
    "phases": {
      "warm": {
        "actions": {
          "shrink" : {
            "number_of_shards": 1
          }
        }
      }
    }
  }
}
```

### freeze

只适用于cold。并且这个action不需要任何参数

> In order to keep indices **available and queryable** for a longer period but at the same time **reduce their hardware requirements** they can be transitioned into a frozen state. 

### delete

只适用于delete phrase。并且这个action不需要任何参数

用于删除index

## 验证

```bash
# 创建ilm策略
$ PUT _ilm/policy/test_policy   
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "set_priority": {
            "priority": 50
          }
        }
      },
      "cold": {
        "min_age": "5m",
        "actions": {
          "set_priority": {
            "priority": 0
          },
          "freeze": {},
          "allocate": {
            "require": {
              "data": "cold"
            }
          }
        }
      },
      "delete": {
        "min_age": "1h",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}

# 关联index模板
$ PUT _template/test_template
{
  "index_patterns": ["test-*"],                 
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "index.lifecycle.name": "test_policy",
    "index.routing.allocation.require.data": "hot"
  }
}

# 创建index
$ PUT test-1
{}
```

## Ref

1. https://www.elastic.co/cn/blog/implementing-hot-warm-cold-in-elasticsearch-with-index-lifecycle-management

