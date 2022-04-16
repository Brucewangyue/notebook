官方文档： https://nacos.io/zh-cn/docs/what-is-nacos.html

Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理

Nacos 的关键特性包括: 

- 服务发现和服务健康监测 
- 动态配置服务 
- 动态 DNS 服务 
- 服务及其元数据管理

## **Nacos** 架构

- **NamingService**: 命名服务，注册中心核心接口 
- **ConfigService**：配置服务，配置中心核心接口

[OpenAPI文档](https://nacos.io/zh­cn/docs/open­api.html )

![image-20220416152651309](assets/image-20220416152651309.png)

## Nacos Server部署

**1.下载源码编译** 

源码下载地址：https://github.com/alibaba/nacos/ 

```sh
cd nacos/
mvn ‐Prelease‐nacos clean install ‐U
cd nacos/distribution/target/
```

**2.下载安装包** 

下载地址：https://github.com/alibaba/Nacos/releases 

### 单机模式部署

[官方文档](https://nacos.io/zh­cn/docs/deployment.html )

```sh
# 解压，进入nacos目录

#单机启动nacos，执行命令
bin/startup.sh ‐m standalone
```

也可以修改默认启动方式

```sh
vi bin/startup.sh
```

![image-20220416153635409](assets/image-20220416153635409.png)

访问nocas的管理端：http://IP:8848/nacos ，默认的用户名密码是 nocas/nocas

![image-20220416153738781](assets/image-20220416153738781.png)

### 集群模式部署

[官网文档](https://nacos.io/zh­cn/docs/cluster­mode­quick­start.html )

**集群部署架构图**

![image-20220416153854489](assets/image-20220416153854489.png)

**步骤**

1. 单机搭建伪集群，复制nacos安装包，修改为nacos8849，nacos8850，nacos8851

2. 以nacos8849为例，进入nacos8849目录

   1. 修改conf\application.properties的配置，使用外置数据源

      ```properties
      #使用外置mysql数据源 
      spring.datasource.platform=mysql 
      ### Count of DB: 
      db.num=1
      ### Connect URL of DB: 
      db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC 
      db.user.0=root
      db.password.0=root
      ```

   2. 将 conf\cluster.conf.example 修改为 cluster.conf ，添加节点配置

      ```properties
      # ip:port 
      192.168.3.14:8849 
      192.168.3.14:8850 
      192.168.3.14:8851
      ```

      nacos8850，nacos8851 按同样的方式配置

3. 创建mysql数据库，sql文件位置：conf\nacos­mysql.sql

4. 修改启动脚本（bin\startup.sh）的jvm参数

   ![image-20220416154459502](assets/image-20220416154459502.png)

5. 分别启动nacos8849，nacos8850，nacos8851

   ```sh
   bin/startup.sh
   ```

6. 官方推荐，nginx反向代理

   ![image-20220416154748306](assets/image-20220416154748306.png)

### prometheus+grafana监控Nacos

https://nacos.io/zh-cn/docs/monitor-guide.html 

Nacos 0.8.0版本完善了监控系统，支持通过暴露metrics数据接入第三方监控系统监控Nacos运行状态

**步骤**

1. nacos暴露metrics数据

   ```properties
   management.endpoints.web.exposure.include=*
   ```

   测试： http://localhost:8848/nacos/actuator/prometheus

   ![image-20220416155119022](assets/image-20220416155119022.png)

2. prometheus采集Nacos metrics数据 

   启动prometheus服务 

   ```sh
    prometheus.exe ‐‐config.file=prometheus.yml
   ```

   测试：http://localhost:9090/graph

   ![image-20220416155218556](assets/image-20220416155218556.png)

3. grafana展示metrics数据

   测试： http://localhost:3000/

   ![image-20220416155233429](assets/image-20220416155233429.png)



## 注册中心

### 注册中心演变及其设计思想



## 配置中心