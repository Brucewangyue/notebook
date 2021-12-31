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

  

