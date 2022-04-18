## Nacos源码编译

源码下载地址： https://github.com/alibaba/nacos/ 

版本： Nacos 1.4.1

1. 进入nacos目录，执行编译命令

   ```sh
   # -P: profiles
   # -D: properties
   # 	‐Drat.skip=true：跳过licensing检查
   #	‐Dmaven.test.skip=true：跳过测试
   # ‐U: 强制刷新本地仓库不存在release版和所有的snapshots版本。
   mvn ‐Prelease‐nacos ‐Dmaven.test.skip=true ‐Drat.skip=true clean install ‐U
   ```

   编译成功后会在distribution/target目录下生成nacos-server-1.4.1.tar.gz

   ![image-20220418105559373](assets/image-20220418105559373.png)

2. 配置数据库

   创建nacos_config数据库（distribution/conf/nacos­mysql.sql）

   在application.properties中开启mysql配置 

   ```properties
   ### If use MySQL as datasource: 
   spring.datasource.platform=mysql 
   ### Count of DB: 
   db.num=1 
   ### Connect URL of DB: 
   db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&soc ketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC 
   db.user.0=root 
   db.password.0=root
   ```

   

3. 启动nacos

   进入console模块，找到启动类 com.alibaba.nacos.Nacos，执行main方法

   ![image-20220418105711772](assets/image-20220418105711772.png)

4. 进入http://localhost:8848/nacos，用户名和密码默认nacos