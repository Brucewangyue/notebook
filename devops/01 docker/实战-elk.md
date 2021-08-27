# 实战 -elk

## 单机搭建

**安装es**

```sh
[root@n1 /]# docker run -d --name elastic1 -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node"  elasticsearch

[root@n1 /]# docker ps
CONTAINER ID   IMAGE           COMMAND                  CREATED          STATUS          PORTS                                                                                  NAMES
bb6800253ccf   elasticsearch   "/docker-entrypoint.…"   11 seconds ago   Up 6 seconds    0.0.0.0:9200->9200/tcp, :::9200->9200/tcp, 0.0.0.0:9300->9300/tcp, :::9300->9300/tcp   elastic1

# 发现启动elasticsearch后,系统变得很卡
# docker stats   # 查看容器资源使用统计
CONTAINER ID   NAME       CPU %     MEM USAGE / LIMIT     MEM %     NET I/O       BLOCK I/O       PIDS
bb6800253ccf   elastic1   0.22%     1.307GiB / 1.748GiB   74.77%    1.05kB / 0B   1.21GB / 41kB   31

# 发现服务器总共的1.748GiB内存被使用了1.307GiB

# 增加内存配置后再创建容器
# -e ES_JAVA_OPTS="-Xms64m -Xmx512m" 内存最小64m 最大512m
[root@n1 /]# docker run -d --name elastic1 -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" -e ES_JAVA_OPTS="-Xms64m -Xmx512m"  elasticsearch

# docker stats
CONTAINER ID   NAME       CPU %     MEM USAGE / LIMIT     MEM %     NET I/O         BLOCK I/O        PIDS
179b1afdeafe   elastic1   0.24%     270.9MiB / 1.748GiB   15.14%    1.61kB / 771B   197MB / 55.3kB   33

[root@n1 /]# curl localhost:9200
{
  "name" : "6OAqTZm",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "lHMfvHR7S4Oq00M_NN_uNg",
  "version" : {
    "number" : "5.6.12",
    "build_hash" : "cfe3d9f",
    "build_date" : "2018-09-10T20:12:43.732Z",
    "build_snapshot" : false,
    "lucene_version" : "6.6.1"
  },
  "tagline" : "You Know, for Search"
}

```

**安装kibana**

