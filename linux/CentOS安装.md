# CentOS

## 版本

- centos7.6 (1810)
- 



## 安装

- 工具：Wmware Workstation Pro 16
- 系统版本： centos8



## 基础配置

- 修改hostname

  ```sh
  hostnamectl set-hostname {主要服务名}-node{序号}-m/s{序号}
  ```

- 安装常用工具

  ```sh
  # lrzsz 是一个上传下载工具，有了它就不需要ftp, 直接在window系统拖动文件到xshell
  # nmcli 是一个比ipconfig更直观显示网卡的工具
  yum install -y wget vim net-tools lrzsz 
  ```

- 安装jdk

  ```sh
  yum install -y java-1.8*
  
  # 查看java安装包，定位jdk安装路径
  rpm -qa | grep java
  
  # 配置JAVA_HOME
  vim /etc/profile
  
  export JAVA_HOME=/usr/lib/jvm/jdk安装包
  export PATH=$PATH:$JAVA_HOME/bin
  
  # 生效配置
  source /etc/profile
  
  # 测试 
  java -version
  ```
## 优化

### 网络

- 配置静态IP

  ```sh
  # 虚拟机里的网卡一般都叫 ifcfg-ensXX，如果是云主机叫 ifcfg-eth0，物理服务器比如戴叫 em1
  # 删除UUID，克隆会出问题
  vi /etc/sysconfig/network-scripts/ifcfg-ens33
      
  TYPE=Ethernet
  BOOTPROTO=static
      
  PROXY_METHOD=none
  BROWSER_ONLY=no
  DEFROUTE=yes
  IPV4_FAILURE_FATAL=no
  IPV6INIT=yes
  IPV6_AUTOCONF=yes
  IPV6_DEFROUTE=yes
  IPV6_FAILURE_FATAL=no
  IPV6_ADDR_GEN_MODE=stable-privacy
  ONBOOT=yes
      
  NAME=ens33
  DEVICE=ens33
  PREFIX=24
      
  GATEWAY=10.0.0.2
  IPADDR=10.0.0.30
  NETMASK=255.255.255.0
  DNS1=10.0.0.2
  # 阿里云
  DNS2=233.5.5.5
  ```

- 关闭防火墙（测试环境）

  ```sh
  systemctl stop firewalld && systemctl disable firewalld
  ```

- 关闭NetworkManager, 使用network-scripts替代

  NetworkManager：如果IP地址不固定，每次重启会修改IP地址

  ```sh
  yum install network-scripts -y 
  systemctl stop NetworkManager && systemctl disable NetworkManager
  ```

- 关闭SELINX

  ```sh
  
  ```

### yum源 

- 备份

  ```sh
  mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
  ```

  

- 替换源

  ```sh
  wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
  ```

  

- yum源数据缓存

  ```sh
  yum clean all && yum makecache
  ```

  





