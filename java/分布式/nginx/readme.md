# nginx

### 安装 tengine

##### 设置为系统服务



### 安装测试服务器 tomcat

创建文件

```
vim /lib/systemd/system/nginx.service


[Unit]
Description=The nginx HTTP and reverse proxy server
After=syslog.target network.target remote-fs.target nss-lookup.target
 
[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
 
[Install]
WantedBy=multi-user.target
————————————————
原文链接：https://blog.csdn.net/nimasike/article/details/51889171
```

修改文件权限

```
chmod 745 nginx.service 
```



### 配置负载均衡（反向代理）





### session 共享

注意时间同步问题：ntpd 服务

```
systemctl status ntpd
```





### 动静分离

##### 首先了解location指令

规定

前缀字符串匹配时，是root/alias 拼接上 前缀字符串来查找最终文件



查找顺序

首先将URI与带有前缀字符串的位置进行比较。然后，它使用正则表达式搜索位置。

正则表达式的优先级更高，除非使用 ^~ 修饰符。在前缀字符串中，NGINX Plus选择最特定的一个(即最长和最完整的字符串)。选择位置处理请求的确切逻辑如下所示: