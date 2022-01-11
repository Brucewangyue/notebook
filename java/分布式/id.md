# 分布式ID

全局唯一，局部唯一，按情况保持递增



## 为什么需要分布式ID

有一个订单表，当数据量大了，需要对



## 锁的实现方案

- UUID
- DB数据库多种模式 (自增主键、segment)
- Redis
- Zk
- Etcd
- SnowFlake
- 美团Leaf (DB-Segment、Zk+SnowFlake)
- 百度uid-generator



## UUID

UUID长度128bit，32个16进制字，占用16个字节的空间，占用空间多，且生成的ID是无序的。对于InnoDB这种聚集主键类型的引擎来说，数据会按照主键进行排序，由于UUID的无序性，InnoDB会产生巨大的IO压力，此时不适合使用UUID做物理主键，可以把它作为逻辑主键，物理主键依然使用自增ID

**需要学习B+树原理才能理解为什么不能直接使用UUID作为物理主键**

- 组成部分：
  为了保证UUID的唯一性，规范定义了包括网卡MAC地址，时间戳，名字空间，随机或伪随机数，时序等元素

- 优点：
  性能非常高，本地生成，没有网络消耗。

- 缺点：
  不易于存储：UUID太长，16字节128位，通常以36长度的字符串表示，很多场景不适用

  信息不安全：基于MAC生成UUID的算法可能造成MAC地址泄露，这个漏洞曾被用于寻找梅丽莎病毒的制作者位置

- 版本

  - Version 1：基于时间的UUID

    基于时间的UUID通过计算当前时间戳、随机数和机器MAC地址得到。由于在算法中使用了MAC地址，这个版本的UUID可以保证在全球范围的唯一性。但与此同时，使用MAC地址会带来安全性问题，这就是这个版本UUID受到批评的地方。如果应用只是在局域网中使用，也可以使用退化的算法，以IP地址来代替MAC地址－－Java的UUID往往是这样实现的（当然也考虑了获取MAC的难度）

  - Version 2：DCE安全的UUID

    DCE（Distributed Computing Environment）安全的UUID和基于时间的UUID算法相同，但会把时间戳的前4位置换为POSIX的UID或GID。这个版本的UUID在实际中较少用到。

  - Version 3：基于名字的UUID（MD5）

    基于名字的UUID通过计算名字和名字空间的MD5散列值得到。这个版本的UUID保证了：相同名字空间中不同名字生成的UUID的唯一性；不同名字空间中的UUID的唯一性；相同名字空间中相同名字的UUID重复生成是相同的。

  - Version 4：随机UUID

    根据随机数，或者伪随机数生成UUID。这种UUID产生重复的概率是可以计算出来的，但随机的东西就像是买彩票：你指望它发财是不可能的，但狗屎运通常会在不经意中到来。

    建议是最好不用（虽然它是最简单最方便的）

  - Version 5：基于名字的UUID（SHA1）

    和版本3的UUID算法类似，只是散列值计算使用SHA1（Secure Hash Algorithm 1）算法

- 建议
  - Version 1/2适合应用于分布式计算环境下，具有高度的唯一性
  - Version 3/5适合于一定范围内名字唯一，且需要或可能会重复生成UUID的环境下
  - 通常我们建议使用UUID来标识对象或持久化数据，但以下情况最好不使用UUID
    - 映射类型的对象。比如只有代码及名称的代码表
    - 人工维护的非系统生成对象。比如系统中的部分基础数据
    - 对于具有名称不可重复的自然特性的对象，最好使用Version 3/5的UUID。比如系统中的用户。如果用户的UUID是Version 1的，如果你不小心删除了再重建用户，你会发现人还是那个人，用户已经不是那个用户了。（虽然标记为删除状态也是一种解决方案，但会带来实现上的复杂性）



## DB数据库多种模式 (自增主键)

**不用**

- 实现原理:
  基于MySQL，最简单的方法是使用auto_increment 来生成全局唯一递增ID，但最致命的问题是在高并发情况下，数据库压力大，DB单点存在宕机风险

- 优点:

  实现简单、基于数据库底层机制

- 缺点:

  高并发情况下，数据库压力大，DB单点存在宕机风险

- 示例：

  这里使用MyISAM引擎来提高插入数据，查询速度，因为也不需要事务的支持

  ```sql
  create table `seq_id`(
  	`id` bigint NOT NULL auto_increment,
  	`value` varchar(1) DEFAULT NULL,
      primary key (`id`)
  ) ENGINE=MyISAM DEFAULT CHARSET=utf8mb4 COMMENT='顺序自增id表'
   
  -- 获取id
  insert into seq_id value();select LAST_INSERT_ID();
  ```



## Redis







## Zookeeper

原理: 利用zookeeper中的顺序节点的特性,制作分布式的序列号生成器(ID生成器)

![image-20220107112832620](assets/image-20220107112832620.png)



## Etcd





## 基于DB多主模式方案（Segment）

**不用**

上面的单DB存在宕机和性能问题，Flickr团队在2010年撰文介绍的一种主键生成策略（Ticket Servers: Distributed Unique Primary Keys on the Cheap ）

- 实现原理

  在分布式系统中我们可以多部署几台机器，每台机器设置不同的初始值，且步长和机器数相等。比如有两台机器。设置步长step为2，TicketServer1的初始值为1（1，3，5，7，9，11…）、TicketServer2的初始值为2（2，4，6，8，10…）

  ![image-20220105152105366](assets/image-20220105152105366.png)

  ```sql
   create table `segment_id`(
  	`id` bigint NOT NULL auto_increment,
  	`max_id` bigint NOT NULL comment '当前最大id',
      `step` bigint NOT NULL comment '号段步长',
      `biz_code` varchar(16) NOT NULL comment '业务类型编码',
      `version` bigint NOT NULL comment '乐观锁',
      primary key (`id`)
   ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='顺序自增id表' auto_increment 1
   
   -- biz_code ：每一个biz_code，max_id都是从0开始，增加了扩展性
  ```

  - Proxy
    - 实现原理：每次向db申请一个号段，加载到内存中，然后采用自增的方式来生成id，这个号段用完后，再次向db申请一个新的号段，这样对db的压力就减轻了很多，同时内存中直接生成id。向数据库申请新号段，对max_id字段做一次update操作，update max_id= max_id + step，update成功则说明新号段获取成功，新的号段范围是(max_id ,max_id +step]。
    - 实现过程：假设有两台Proxy
      - 客户端请求分布式ID，Proxy根据biz_code，max_id申请号段，这里使用乐观锁防止号段混乱
      - Proxy如果挂了，重启启动后，如果之前申请的号段还有未使用的直接抛弃，重新申请号段

- 优点:

  利用了缓存，减轻DB压力，性能提升

- 缺点:

  依然存在DB模式下的性能瓶颈，ID最大值的限制



## 雪花算法

举一个生活中的例子：身份证

- 省份编码是全局唯一
- 区域编码是局部唯一
- 时间是递增数
- 随机数某一天最大是9999，表示某个区域某一天不超过9999人出生

![image-20211231151233854](assets/image-20211231151233854.png)

那么，演化到分布式ID中：

![image-20211231151742052](assets/image-20211231151742052.png)

这里的分布式ID是模拟的，在计算机中，二进制位如果只使用18个长度，那么只能表示2^18次方个ID，那么显然不够，所以使用的是64位的Long类型来实现，排除第一号位是符号位，那么可以生成2^63次方个ID



## 美团Lead

[官方文档](https://github.com/Meituan-Dianping/Leaf)

### 安装

- 安装工具

  ```sh
  yum install -y git maven
  ```

- 克隆项目

  ```sh
  cd /usr/local
  git clone https://github.com/Meituan-Dianping/Leaf.git
  ```

- 配置maven源

  ```sh
  vim /etc/maven/setting.xml
  
  # mirrors节点
   <mirror>
           <id>aliyun-maven</id>
           <mirrorOf>*</mirrorOf>
           <name>aliyun-maven</name>
           <url>http://maven.aliyun.com/nexus/content/groups/public</url>
  </mirror> 
  ```

- maven构建项目

  ```sh
  cd /usr/local/Leaf
  mvn clean install -DskipTests
  ```


### 运行

- 启动：

  ```sh
  cd leaf-server
  # 方式一：maven
  mvn spring-boot:run
  # 方式二：shell
  sh deploy/run.sh
  ```

- 测试

  ```sh
  #segment
  curl http://localhost:8080/api/segment/get/leaf-segment-test
  #snowflake
  curl http://localhost:8080/api/snowflake/get/test
  ```

  

