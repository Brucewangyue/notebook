# Redis

[中文文档](http://redis.cn/)

redis与内核之间使用的是epoll(多路复用)技术

![image-20211216103412448](assets/image-20211216103412448.png)

## 简介

​	**常识一：数据访问的两个角度**

- 磁盘
  - 寻址（速度）：ms  
  - 带宽（单位时间可以传输多少数据）：G/M
- 内存
  - 寻址：ns
  - 带宽：

​	秒>毫秒>微妙>纳秒：可以看出内存比磁盘快了10万倍

​	**常识二：I/O buffer：成本问题**

​	磁盘与磁道，扇区，一扇区 512Byte带来一个成本变大：索引 ，这样索引就不能像现在这样用4位来表示了，所以一次读取的是4K ，操作系统无论你读多少，都是最少4k从磁盘拿

​	从以上常识可以得知：随着文件变大，读取速度变慢，这时就产生了数据库，用于提高读取速度。关系型数据库建表的时候必须给出schema类型，有了类型就定死了字节宽度，存的时候倾向于行级存储，假如插入一行，其中某个字段为空，内存空间还是会开辟占位，未来修改的时候内存就不用移动位置

![image-20211215164724822](assets/image-20211215164724822.png)

​	4K不是一定的，取决于上层应用的需求，一般家用会把磁盘格式化成4k的，而倾向于存储视频那种大数据，可以格式化成更大的数据页

​	关系型数据库，表很大，性能下降？：

- 如果表有索引，增删改变慢
- 查询速度
  - 1个或少量查询依然很快
  - 并发大的时候会受硬盘带宽影响速度

​	**常识三：数据在磁盘和内存体积不一样**

​	SAP HANA内存级别的关系型数据库2T，价格在2亿，太贵，只有500强在用，那么就出现了折中方案，redis，memcached 

当然出现这么多的问题，主要是受制于当前的两个基础设施

- 冯诺依曼体系的硬件
- 以太网，tcp/ip的网络



## 技术选型

[技术选型网站](https://db-engines.com/en/)

**这个世界有3种数据表示方式：**

- k=a k=1
- k=[1,2,3]  k=[a,x,f]
- k={x=y} k=[{},{}]



## Redis 和 Memcached 的区别

​	memcached 没有类型，会导致一些网络IO的问题

- 因为没有类型，memcached 要返回value所有的数据到client，假如这个value是一个数组，或者对象，那么就要在client自己实现代码去解码
- redis的server中对每种类型都有自己的方法，如：index()、lpop()，服务端会直接把一个很小的结果返回client

​	redis的优势：计算向数据移动



	## 安装

[官网](https://redis.io/)

### 一：源码安装

centos 6.x，redis 官网5.x

```sh
yum install wget
cd ~
mkdir soft
cd soft
wget    http://download.redis.io/releases/redis-5.0.5.tar.gz
tar xf    redis...tar.gz
cd redis-src

#（看README.md）

# make  linux 编译工具，需要 makefile 才知道如何编译
yum install  gcc  
make distclean
make
cd src   ....生成了可执行程序

# 服务化
cd ..
make install PREFIX=/opt/redis5

# 添加环境变量
vi /etc/profile
...   export  REDIS_HOME=/opt/redis5
...   export PATH=$PATH:$REDIS_HOME/bin

source /etc/profile

cd utils
./install_server.sh  （可以执行一次或多次）    
	a)  一个物理机中可以有多个redis实例（进程），通过port区分
	b)  可执行程序就一份在目录，但是内存中未来的多个实例需要各自的配置文件，持久化目录等资源
	c)  service   redis_6379  start/stop/stauts     >   linux   /etc/init.d/****
	d)脚本还会帮你启动！17,ps -fe |  grep redis  
```

### 二：docker安装

### 三：k8s安装



## redis-cli

[命令大全](http://doc.redisfans.com/)

### FLUSHDB

​	清空当前数据库中的所有 key。生产环境要禁用（改名）

#### Object

```sh
# 查看key的真实类型
object encoding key

# embstr: 字符串
# raw:
# int: 数值
```

#### STRLEN

```sh
set k1 hello
STRLEN k1
# 结果是5
set k2 9
STRLEN k2
# 1
# 常识：1字节=8位 可以标识 -128~127
set k3 中
STRLEN k3
# 3
get k3
# "\xe4\xb8\xad"
# 因为是3个字节所以显示的是3

# 二进制安全：字节流，字符流
# redis 拿的是字节流
# 因为redis 面向的是任何客户端，很多客户端对字节的宽度理解不一，只要是同一种客户端存或者取那么就没有问题
# 比如我有一个java程序用的是UTF-8编码，那么汉字长度就是3个字节，如果有另外一个程序用的GBK编码，那么汉字长度是2个字节

# 如果用 redis-cli --raw 来连接redis，那么get key 的时候，如果value符合当前客户端编码规则，那么会转码，比如对于上面的 get k3 会得到 中
# 如果我这时的客户端用了gbk编码格式来连接redis，那么get k3 会乱码
```

​	常识：字符集：ascii ，其他一般叫扩展字符集？？ todo 补充 ascii知识

**结论：客户端需要沟通好使用相同的编码，否则有可能出现乱码**





## 类型

```json
key:{
    type:"string",  // redis的类型
    encoding:"int"  // 这个类型提升计算（incr...）
}
```



### 一：String

​	value的类型：

- 字符串
- 数值
- 位图



#### 字符串

**NX**

```sh
set k v nx
```

​	key不存在的时候设置值

**应用的场景：分布式锁**

**XX**

```sh
set kv xx
```

​	key存在的时候设置值



**实战一：图形验证码的文本**



**实战二：分布式锁**

命令 `SET resource-name anystring NX EX max-lock-time` 是一种在 Redis 中实现锁的简单方法。

客户端执行以上的命令：

- 如果服务器返回 `OK` ，那么这个客户端获得锁。
- 如果服务器返回 `NIL` ，那么客户端获取锁失败，可以在稍后再重试。

设置的过期时间到达之后，锁将自动释放。

可以通过以下修改，让这个锁实现更健壮：

- 不使用固定的字符串作为键的值，而是设置一个不可猜测（non-guessable）的长随机字符串，作为口令串（token）。
- 不使用 [*DEL*](http://doc.redisfans.com/key/del.html#del) 命令来释放锁，而是发送一个 Lua 脚本，这个脚本只在客户端传入的值和键的口令串相匹配时，才对键进行删除。

这两个改动可以防止持有过期锁的客户端误删现有锁的情况出现。

以下是一个简单的解锁脚本示例：

```lua
if redis.call("get",KEYS[1]) == ARGV[1]
then
    return redis.call("del",KEYS[1])
else
    return 0
end
```





**实战三：用户登录状态**

todo  完善



#### 数值 int

**实战一：秒杀**



**实战二：详情页**

​	incr 库存



#### 位图 bitmap

![image-20211216111445594](assets/image-20211216111445594.png)

```sh
# setbit b1 偏移量 value
# 变异量表示的是位的偏移量

setbit b1 7 1
strlen b1 
# 1
setbit b2 8 1
strlen b2 
# 2   因为有9位，一个字节8位，redis看的是字节流，所以这里是2
```



**实战**一：有用户系统，统计用户登录天数，且窗口随机

​	做法一：在数据库创建一张表，每一个用户登录就记录一条

​	这个表的设计，用户id列需要几个字节（假设4个），如果是uuid那就更多了，日期列也需要4个字节，那么一次登录就要消耗8个字节，有200天登录了就是1600个字节

​	如果一个系统特别大，用户特别多，登录特别频繁，加入是京东，这个怎么优化？

​	做法二：使用redis

​	用一个bit的每一位代表一个用户一年的某一天是否登录，登录了那这个位就是1

![image-20211216120739824](assets/image-20211216120739824.png)

```sh
# sean在第2天登录了
setbit userlogin:sean 1 1
# sean在第8天登录了
setbit userlogin:sean 7 1
# sean 在第365天登录了
setbit userlogin:sean 364 1
# 记录一年一个用户的登录次数只需要 365/8 个字节 40几个字节
STRLEN userlogin:sean
# 查询倒数16天sean的登录次数
BITCOUNT userlogin:sean -2 -1
```

**实战**二：京东就是你们的，618做活动：送礼物，大库备货多少礼物假设京东有2E用户

僵尸用户，冷热用户/忠诚用户

需求：活跃用户统计！随即窗口，比如说 1号~3号 连续登录要   去重

![image-20211216120751774](assets/image-20211216120751774.png)

```sh
# key表示日期，位表示用户（这里要给用户和位做好一个映射关系）
# 1号用户在20190101登录了
setbit 20190101   1  1
setbit 20190102   1  1
# 7号用户在20190102登录了
setbit 20190102   7  1
# bitop位运算，计算的结果放入destkey
# 统计 20190101 20190102 这两天有哪些人登录了
bitop  or  destkey 20190101  20190102
# 查看一共有多少人登录了
BITCOUNT  destkey  0 -1 
```



### 二：List 

​	链表，key中包含头尾指针，并且维护了正反向索引

​	List 有顺序，可重复

![image-20211216141851547](assets/image-20211216141851547.png)

​	redis对 list 类型提供的一些方法让 list 能轻松的扩展类型

- 栈 （先进后出）：lpush lpop / rpush rpop 同向命令 

  - ```sh
    # 插入后的顺序是 d c b a
    lpush a b c d
    ```

- 队列（先进先出）：lpush rpop / rpush lpop 反向命令

- 数组：lindex lset linsert lrem llen

- 单播订阅：

  - ```sh
    # 订阅key，直到有消息才弹出。0表示一直等待
    blpop key 0 
    ```



### 三：hash 

​	一般存放一个简单的只有一级深度的键值对 ，因为他不支持json那种无限层级的属性键值对



### 四：Set

​	不可重复，无序

- 交集 SINTER SINTERSTORE
- 差集 SDIFF 通过key的顺序控制差集
- 并集 SUNION SUNIONSTORE



**SRANDMEMBER** 

​	如果命令执行时，只提供了 `key` 参数，那么返回集合中的一个随机元素。

- 如果 `count` 为正数，且小于集合基数，那么命令返回一个包含 `count` 个元素的数组，数组中的元素**各不相同**。如果 `count` 大于等于集合基数，那么返回整个集合。
- 如果 `count` 为负数，那么命令返回一个数组，数组中的元素**可能会重复出现多次**，而数组的长度为 `count` 的绝对值。

**SPOP**

​	将随机元素从集合中移除并返回

**实战一：抽奖**



**实战二：家庭争斗**



### 五：Sorted Set

​	为什么它的命令是Z开头？因为S被占用了，直接取的26个字母的最后一个

​	支持对元素排序，没有分值默认按值的字符串排，物理存储方式是按数值左小右大

![image-20211216153845425](assets/image-20211216153845425.png)



**面试题：排序是如何实现的？速度？**

​	skip list 跳表  todo 深入学习

![image-20211216161726029](assets/image-20211216161726029.png)

**实战一：排行榜**



## 高级扩展



### 模块

[模块列表](https://redis.io/modules)

#### 布隆过滤器 

​	解决缓存穿透问题，布隆过滤器首先知道你有啥，没有的也不去查询数据库。那么如果数据库的数据特别多，布隆过滤器要加载的数据就会很多，它是如何用小空间存大数据的？

[官方文档](https://github.com/RedisBloom/RedisBloom)

​	通过源码安装

```sh
wget https://github.com/RedisBloom/RedisBloom/archive/refs/heads/master.zip

# 解压需要安装unzip
unzip 文件名

# 进入文件夹，有makefile的目录 
# 编译需要安装gcc

make

# 拷贝编译好的.so后缀的程序到/opt/redis/modules 

# 启动
redis-server --loadmodule /opt/redis/modules/redisbloom.so redis.conf
```



## 缓存的几大问题

### 穿透

​	一般情况，用户首先查询redis，redis没有查询数据库。但是会存在一个问题，就是用户查询的这条数据数据库也没有，那么用户端就会不停的访问数据库，让数据库造成无用的性能损耗

### 雪崩

