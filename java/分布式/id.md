# 分布式ID



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

  