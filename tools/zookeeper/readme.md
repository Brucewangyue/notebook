# Zookeeper

## 简介

Zookeeper是一个分布式的、开源的程序协调服务，是 [hadoop](https://so.csdn.net/so/search?q=hadoop) 项目下的一个子项目。

他提供的主要功能包括：配置管理、名字服务、[分布式锁](https://so.csdn.net/so/search?q=分布式锁)、集群管理 。

## ZNode

Zookeeper 底层是一套数据结构。这个存储结构是一个**树形**结构，其上的每一个节点，我们称之为“znode”，每一个 znode 默认能够存储 1 MB的数据，其中包括数据更改的版本号、acl更改。统计结构也有时间戳。版本号和时间戳允许ZooKeeper验证缓存并协调更新。znode的数据每次更改，版本号就会增加。例如，每当客户端检索数据时，它也会接收到数据的版本。当客户机执行更新或删除操作时，它必须提供它正在更改的znode数据的版本

- **Persistent 持久化节点**: 在节点创建后，就一直存在，直到有删除操作来主动清除这个节点。否则不会因为创建该节点的客户端会话失效而消失

- **Persistent_Sequential**: 这类节点的基本特性和上面的节点类型是一致的。额外的特性是，在 ZK 中，每个父节点会为他的第一级子节点维护一份时序，会记录每个子节点创建的先后顺序。（基于这个特性，在创建子节点的时候，可以设置这个属性，那么在创建节点过程中，ZK 会自动为给定节点名加上一个数字后缀，作为新的节点名。这个数字后缀的范围是整型的最大值。在创建节点的时候只需要传入节点 “/test_”，这样之后，zookeeper 自动会给”test_”后面补充数字）

  ```sh
  create -s
  ```

  

- **Ephemeral 临时节点**: 临时节点的生命周期和客户端会话绑定。也就是说，如果客户端会话失效，那么这个节点就会自动被清除掉

  ```sh
  create -e
  ```

- **Ephemeral_Sequential 临时顺序节点**: 此类节点属于临时节点，不过带有顺序，客户端会话结束节点就消失。

- **Watches**: 客户端可以设置对znode的事件监听，当触发了事件，zk会发送给客户端一个通知

-  **Access Control List (ACL)** : 每个节点都有ACL，限制谁能做什么

- **TTL Nodes**: 当创建PERSISTENT或PERSISTENT_SEQUENTIAL znode时，您可以选择为znode设置一个以毫秒为单位的TTL。如果znode在TTL内没有被修改，并且没有子节点，它将成为服务器在将来某个时候删除的候选节点（TTL节点必须通过系统属性启用，因为它们在默认情况下是禁用的）



## 安装

​	1主2从环境搭建，版本3.5.9

```sh
# 安装jdk
# 下载 https://zookeeper.apache.org/releases.html
# 解压
# 配置
# 在目录配置项中指定的目录创建myid文件，文件中写上id
# 启动
```



## 使用

[官方Getting Started Guide v3.7.0](https://zookeeper.apache.org/doc/r3.7.0/zookeeperStarted.html#sc_ConnectingToZooKeeper)

- 创建循序节点

  ```sh
  # 创建一个根znode
  create /ids
  # 创建顺序znode
  create -s /ids//biz_
  
  reated /ids/biz_0000000000
  
  # 删除顺序节点
  delete /ids/biz_0000000008
  
  # 再次创建顺序znode
  create -s /ids//biz_
  
  reated /ids/biz_0000000009
  # 结论：已创建过的顺序znode被删除，再次创建会跳过被删除的znode
  ```

  