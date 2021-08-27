# Docker-Compose

## 概述

**单机版容器编排**

[概述](https://docs.docker.com/compose/)

## 安装

### 测试

```sh
# docker-compose up
Creating network "composetest_default" with the default driver
Building web
Attaching to composetest_redis_1, composetest_web_1

# 首先会创建一个网络，所有的服务（容器）将会使用该网络，并且所有的服务可直接通过服务名进行访问，如果某个服务有多个容器实例，则根据服务名自动实现负载
# 可以看到创建的容器命名规则：目录名_服务名_序号
```

