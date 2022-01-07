# edtc

[官方文档](https://etcd.io/docs/v3.5/) 



##  搭建集群

一主二从

- 系统版本：centos 7.26 1801
- etcd版本：3.4.18
- 硬件要求：

```sh
# 设置hostname、静态ip
# node1 hostname:etcd-n01-main   ip:10.0.0.30
# node2 hostname:etcd-n02-follwer   ip:10.0.0.31
# node3 hostname:etcd-n03-follwer   ip:10.0.0.32

# 安装
yum install etcd -y

# 集群配置
cp /etc/etcd/etcd.conf /etc/etcd/etcd.conf.bak
vi /etc/etcd/etcd.conf

ETCD_LISTEN_PEER_URLS="http://10.0.0.30:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.0.0.30:2379,http://127.0.0.1:2379"
ETCD_NAME="Main"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.0.0.30:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.30:2379"
ETCD_INITIAL_CLUSTER="Main=http://10.0.0.30:2380,Follwer01=http://10.0.0.31:2380,Follwer02=http://10.0.0.32:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"

# 防火墙
# 如果开启防火墙，需要开放端口
firewall-cmd --zone=public --add-port=2379/tcp --permanent
firewall-cmd --zone=public --add-port=2380/tcp --permanent
firewall-cmd --reload
# 如果是云主机，需要在安全规则添加端口欧

# 重启服务
systemctl restart etcd

# 查看集群成员
etcdctl member list

# 验证
# 在节点一运行
etcdctl put test_key test_value
# 在其他节点运行
etcdctl get test_key
```



**通过二进制文件安装**

[官方文档](https://etcd.io/docs/v3.5/install/)

[配置文件模板](https://github.com/etcd-io/etcd/blob/release-3.4/etcd.conf.yml.sample)

- 系统版本：centos 8
- etcd版本：3.4.18

```sh
# centos 8 无法直接通过yum来安装
# 下载
# 解压
# 拷贝 etcd etcdctl 到 /usr/local/bin
# 创建配置文件 - 不需要的配置可以删除，etcd都有默认值
# 创建配置文件中指定的data数据目录、wal日志目录
# 启动
etcd --config-file etcd.conf
# 测试
```





查看key的属性

```json
etcdctl get test_key -w json

{
	"header":{
	    "cluster_id":12312,
        "member_id":"",
        // 全局唯一自增id (64位)，集群内的任何一个操作都会使其自增
        "revision":49,
        // leader任期：全局自增，每次选择选举都会使其自增
        "raft_term":8,
	},
    "kvs":[
        {
            "key":"a2V5",
            // 该key被创建时的revision
            "create_revision":47
            // 该key被修改时的revision
            "mod_revision":49,
            // 自身的版本记录（B+树）
            "version":3,
            "value":"bWFyaw=="
        }
    ],
    "count":1
}
```



## 使用

- 通过前缀查询

  ```sh
  etcdctl get --prefix /lock/test
  ```

  

