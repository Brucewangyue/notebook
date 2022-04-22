## Spring Cloud整合Dubbo

### Provider端

1. 引入依赖

   ```xml
   <dependency> 
       <groupId>com.alibaba.cloud</groupId> 
       <artifactId>spring‐cloud‐starter‐dubbo</artifactId> 
   </dependency> 
   <dependency> 
       <groupId>com.alibaba.cloud</groupId> 
       <artifactId>spring‐cloud‐starter‐alibaba‐nacos‐discovery</artifactId> 
   </dependency>
   ```

2. application.yml

   ```yaml
   dubbo:
     scan:
     	# 指定 Dubbo 服务实现类的扫描基准包
       base-packages: com.va.order.service
     protocol:
     	# dubbo 协议
       name: dubbo
       # dubbo 协议端口（ ‐1 表示自增端口，从 20880 开始）
       port: -1
   spring:
     main:
       allow-bean-definition-overriding: true
     application:
       name: server-provider
     cloud:
       nacos:
         discovery:
           server-addr: 127.0.0.1:8848
   ```

3. **暴露服务** 服务实现类上配置@DubboService

   ```java
   @DubboService 
   public class UserServiceImpl implements UserService { 
       @Autowired 
       private UserMapper userMapper; 
       @Override 
       public List<User> list() { 
           return userMapper.list(); 
       } 
       @Override 
       public User getById(Integer id) { 
           return userMapper.getById(id); 
       } 
   }
   ```

### Consumer端

1. 引入依赖

   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-web</artifactId>
   </dependency>
   <dependency> 
       <groupId>com.alibaba.cloud</groupId> 
       <artifactId>spring‐cloud‐starter‐dubbo</artifactId> 
   </dependency> 
   <dependency> 
       <groupId>com.alibaba.cloud</groupId> 
       <artifactId>spring‐cloud‐starter‐alibaba‐nacos‐discovery</artifactId> 
   </dependency>
   ```

2. application.yml

   ```yaml
   dubbo:
     cloud:
     	# 服务订阅
       subscribed-services: server-provider
     protocol:
       name: dubbo
       port: -1
   spring:
     main:
       allow-bean-definition-overriding: true
     application:
       name: server-consumer
     cloud:
       nacos:
         discovery:
           server-addr: 127.0.0.1:8848
   ```

3. **引入服务** 服务消费方通过@DubboReference

   ```java
   @RestController 
   @RequestMapping("/user") 
   public class UserConstroller { 
       @DubboReference 
       private UserService userService; 
       
       @RequestMapping("/info/{id}")
       public User info(@PathVariable("id") Integer id){ 
       return userService.getById(id); 
       } 
       
       @RequestMapping("/list") 
       public List<User> list(){ 
           return userService.list(); 
       } 
   }
   ```

   

### 从Open Feign迁移到Dubbo

Dubbo Spring Cloud 提供了方案，即 @DubboTransported 注解，支持在类，方法，属性上使用。能够帮助服务消费端的 Spring Cloud Open Feign 接口以及 @LoadBalanced RestTemplate Bean 底层走 Dubbo 调用（可切换 Dubbo 支持的协 议），而服务提供方则只需在原有 @RestController 类上追加 Dubbo @Servce 注解（需要抽取接口）即可，换言之，在不调整 Feign 接口以及 RestTemplate URL 的前提下，实现无缝迁移

1. 修改Provider

   ```java
   @DubboService 
   @RestController
   @RequestMapping("/user")
   public class UserServiceImpl implements UserService { 
       @Autowired 
       private UserMapper userMapper; 
       
       @RequestMapping("/list")
       @Override 
       public List<User> list() { 
           return userMapper.list(); 
       } 
       
       @RequestMapping("/getById/{id}")
       @Override 
       public User getById(Integer id) { 
           return userMapper.getById(id); 
       } 
   }
   ```

2. 修改Consumer

   1. 添加依赖

      ```xml
       <dependency> 
           <groupId>org.springframework.cloud</groupId> 
           <artifactId>spring‐cloud‐starter‐openfeign</artifactId> 
      </dependency>
      ```

   2. 启动类上添加@EnableFeignClients

   3. feign接口添加 @DubboTransported 注解

      1. RestTemplate

         ```java
         @Bean 
         @LoadBalanced 
         @DubboTransported 
         public RestTemplate restTemplate() { 
             return new RestTemplate(); 
         }
         ```

      2. Feign接口

         ```java
         @FeignClient(value = "spring‐cloud‐dubbo‐provider‐user‐feign",path = "/user") 
         @DubboTransported(protocol = "dubbo") 
         public interface UserDubboFeignService { 
             @RequestMapping("/list")
             public List<User> list(); 
             
             @RequestMapping("/getById/{id}") 
             public User getById(@PathVariable("id") Integer id); 
         } 
         ```