# 实战演练

## Namespace

隔离Pod的访问，主要作用是实现**多套环境的资源隔离**或**多租户的资源隔离**



**默认命名空间**

```sh
default                # 未执行ns的对象都被分配到default下
kube-node-lease        # 集群节点间的心跳维护
kube-public            # 任何人都可以访问的资源
kube-system            # 所有由K8s系统创建的资源都处于这个命名空间
```

**演练**

```sh
# 创建命名空间
$ kubectl create namespace dev

# 查看命名空间
$ kubectl get ns

NAME                   STATUS   AGE
default                Active   2d2h
dev                    Active   7s

# 删除命名空间，并且会自动删除该命名空间下的所有pod，因为pod都是运行在集群的各个节点上，所以删除命名空间是需要等待一段时间
$ kubectl delete ns dev
```



## Pod

pod是k8s的最小操作单元，程序运行在容器中，容器运行在pod中

![image-20210902151023746](assets/image-20210902151023746.png)

Pause容器称为根容器，其他容器称为用户用气

```sh
# k8s的组件都是以pod的形式运行的
$ kubectl get po -n kube-system -o wide
NAME                              READY   STATUS    AGE    IP                NODE
coredns-7f89b7bc75-d9bnt          1/1     Running       2d3h   10.244.2.3        node2  
coredns-7f89b7bc75-vl2sj          1/1     Running       2d3h   10.244.2.2        node2
# k8s基础组件
etcd-master1                      1/1     Running       2d3h   192.168.177.100   master1  
kube-apiserver-master1            1/1     Running       2d3h   192.168.177.100   master1  
kube-controller-manager-master1   1/1     Running       2d3h   192.168.177.100   master1  
kube-scheduler-master1            1/1     Running       2d3h   192.168.177.100   master1
# 网络组件在每个节点都安装
kube-flannel-ds-4ch5v             1/1     Running       2d2h   192.168.177.100   master1
kube-flannel-ds-4df4x             1/1     Running       2d2h   192.168.177.101   node1    
kube-flannel-ds-tldc9             1/1     Running       2d2h   192.168.177.102   node2    
# 代理在每个节点都安装
kube-proxy-84rhw                  1/1     Running       2d3h   192.168.177.100   master1  
kube-proxy-hzs2c                  1/1     Running       2d3h   192.168.177.102   node2    
kube-proxy-j5xtj                  1/1     Running       2d3h   192.168.177.101   node1  
```

**演练**

```sh
# 在dev命名空间下运行pod，如果不指定命名空间默认在default命名空间下运行
$ kubectl run my-nginx --image=nginx -n dev

pod/pod123 created

# 查看已经运行的pod
$ kubectl get pod -n dev -o wide

NAME        READY   STATUS              RESTARTS   AGE   IP           NODE    
my-nginx    1/1     Running             0          93s   10.244.1.6   node1 

# 查看pod详细信息，查看容器创建过程，排错
$ kubectl describe po my-nginx

# 访问pod，使用k8s创建的网卡访问（该IP不稳定，用于测试）
$ curl 10.244.1.6

# 删除pod
$ kubectl delete pod pod123 -n dev

pod "pod123" deleted

# 再次查看已经运行的pod，发现该命名空间下已经没有pod了
# 注意：如果在删除pod的时候有deployment-pod控制器，那么删除pod后，pod控制器又会自动的重新创建一个新的pod，想要删除pod，就要删除pod控制器
```



## Label

- kv键值对

```sh
-----------------------------
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    version: "3.0"
    env: "test"
---
apiVersion: v1
kind: Pod
metadata:
  name: nginxpod
  namespace: dev
spec:
  containers:
  - name: nginx-containers
    image: nginx
-----------------------------
```

**标签选择器 lable selector**

[api文档](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)

- 基于等式
  - name=value
  - name!=value
- 基于集合
  - name in (v1,v2)
  - name notin (v1,v2)

**演练**

```sh
# 根据配置文件创建pod
$ kubectl apply -f pod-nginx.yaml

# 查看ns的label信息
$ kubectl describe ns dev

Name:         dev
Labels:       env=test
              version=3.0
Annotations:  <none>
Status:       Active

# 查看po并显示labels
$ kubectl get po -n dev --show-labels

NAME       READY   STATUS    RESTARTS   AGE   LABELS
nginxpod   1/1     Running   0          31s   <none>

# 添加label
$ kubectl label pod nginxpod -n dev env=test

pod/nginxpod labeled

# 更新label
$ kubectl label pod nginxpod -n dev env=dev --overwrite

# 再创建一个pod 
$ kubectl run nginx1 --image=nginx --port=80 -n dev

# 查看po并显示labels
$ kubectl get po -n dev --show-labels

NAME       READY   STATUS    RESTARTS   AGE     LABELS
nginx1     1/1     Running   0          31s     run=nginx1
nginxpod   1/1     Running   0          6m15s   env=dev

# 筛选
$ kubectl get po -n dev -l "env=dev"

NAME       READY   STATUS    RESTARTS   AGE
nginxpod   1/1     Running   0          7m46s

# 多条件筛选
$ kubectl get po -n dev -l "env in (dev,test)"


# 删除label
$ kubectl label po nginx1 run- -n dev

pod/nginx1 labeled
```



## Deployment

> pod 控制器的一种

在k8s中，pod是最小的控制单元，但是k8s很少直接控制pod，一搬都是通过pod控制器来完成。pod控制器用于pod的管理，确保pod资源符合预期的状态，当pod的资源出现故障时，会尝试进行重启或重建pod

![image-20210902162328826](assets/image-20210902162328826.png)

**演练**

```sh
# 创建pod和deployment控制器
$  kubectl create deployment my-dep --image=nginx --port=80 --replicas=3 -n dev

# 查看结果
$ kubectl get deployment,po -n dev -o wide
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES   SELECTOR
deployment.apps/my-dep   3/3     3            3           50s   nginx        nginx    app=my-dep

NAME                          READY   STATUS    RESTARTS   AGE   IP            NODE  
pod/my-dep-6949894f8c-8ndl9   1/1     Running   0          50s   10.244.2.14   node2
pod/my-dep-6949894f8c-q6h9m   1/1     Running   0          50s   10.244.2.13   node2
pod/my-dep-6949894f8c-xqb4z   1/1     Running   0          50s   10.244.1.10   node1

# 删除控制器
$ kubectl delete deploy my-dep -n dev
```

通过配置文件创建

```sh
-----------------------------
# 命名空间
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    version: "3.0"
    env: "test"
---
# 控制器
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-nginx
  namespace: dev
  labels:
    app: deploy-nginx
spec:
  # 副本数
  replicas: 3
  # 关联pod
  selector:
    matchLabels:
      app: deploy-nginx
  # 创建pod的模板
  template:
    metadata:
      labels:
        # 这里绑定了控制器
        app: deploy-nginx
    spec:
      containers:
      - name: nginx-containers
        image: nginx:latest
        #ports:
        #- containerPort: 80
        #  protocol: TCP
-----------------------------
```

```sh
# 创建控制器和pod
$  kubectl apply -f deploy-nginx.yaml

# 查看结果
$ kubectl get deploy,po -n dev --show-labels -o wide

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS         IMAGES   SELECTOR           LABELS
deployment.apps/deploy-nginx   3/3     3            3           15m   nginx-containers   nginx    app=deploy-nginx   app=deploy-nginx

NAME                                READY   STATUS    RESTARTS   AGE   IP            NODE    LABELS
pod/deploy-nginx-686b9c7f68-ct4wt   1/1     Running   0          15m   10.244.2.19   node2           app=deploy-nginx,pod-template-hash=686b9c7f68
pod/deploy-nginx-686b9c7f68-qm4fs   1/1     Running   0          15m   10.244.1.13   node1           app=deploy-nginx,pod-template-hash=686b9c7f68
pod/deploy-nginx-686b9c7f68-tstss   1/1     Running   0          15m   10.244.2.20   node2         app=deploy-nginx,pod-template-hash=686b9c7f6

# 通过pod ip访问
$ curl 10.244.2.19

# 删除其中的一个pod
$ kubectl delete pod deploy-nginx-686b9c7f68-ct4wt -n dev

# 由于deployment控制器会确保pod资源符合预期的状态，会把删除的pod重新创建
# 再次查看pod时发现还是存在3个pod，并且pod ip已经改变
```



## Service

虽然每个pod都会分配一个pod ip ，然后存在以下问题

- pod ip 会随着pod的重建而改变
- pod ip 仅在集群内可见的虚拟ip，外部无法访问

**service解决以上问题**

service 可以看作时一组同类pod**对外的访问接口**，借助service，应用可以方便的实现服务发现和负载均衡

![image-20210902172924789](assets/image-20210902172924789.png)

**演练**

```sh
# 创建服务 kubectl service deploy [deploy-name] [args]
$ kubectl expose deploy deploy-nginx --name=svc-nginx --type=ClusterIP --port=80 --target-port=80 -n dev

# 查询
$ kubectl get svc -n dev

NAME        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
svc-nginx   ClusterIP   10.106.35.71   <none>        80/TCP    25s

# 通过 cluster ip 访问
$ curl 10.106.35.71

# 此时还是只能在集群内部访问，通过--type=NodePort 让外部访问
$ kubectl expose deploy deploy-nginx --name=svc-nginx1 --type=NodePort --port=80 --target-port=80 -n dev

# 再次查询
$ kubectl get svc -n dev

NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
svc-nginx    ClusterIP   10.106.35.71    <none>        80/TCP         3m39s
svc-nginx1   NodePort    10.100.51.170   <none>        80:31058/TCP   12s

# 此时外部网路可以正常访问，并且实现了负载均衡
http://192.168.177.100:31058/

# 删除
$ kubectl delete svc svc-nginx svc-nginx1 -n dev

service "svc-nginx" deleted
service "svc-nginx1" deleted
```

通过配置文件创建

```sh
-----------------------------
# 服务
apiVersion: v1
kind: Service
metadata:
  name: svc-nginx
  namespace: dev
spec:
  type: ClusterIP
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  # clusterIP: 0.106.35.100
  # 关联deployment
  selector:
      app: deploy-nginx
  
-----------------------------
```







## 部署Redis



a) 创建redis配置文件

```sh
# 将 https://raw.githubusercontent.com/redis/redis/6.0/redis.conf 内容保存到redis.conf
# 不显示注释和空行
$ grep -v "#" redis.conf | grep -v "^$"

# 将没有注释和空行的内容覆盖redis.conf
# 修改redis.conf
$ vim redis.conf

bind 0.0.0.0
dir /data
logfile "/temp/redis.log"
```

b) 将redis配置文件存放在configMap

```sh
$ kubectl create cm redis-single-conf --from-file redis.conf

# 查看 
$ kubectl edit cm redis-single-conf

apiVersion: v1
data:
  # 类文件键 ，被挂载后 redis.conf 会作为文件名
  redis.conf: | 
    bind 0.0.0.0
    protected-mode yes
    port 6379
    tcp-backlog 511
    timeout 0
    tcp-keepalive 300
    daemonize no
    supervised no
    pidfile /var/run/redis_6379.pid
    loglevel notice
    logfile "/temp/redis.log"
    databases 16
    always-show-logo yes
    save 900 1
    save 300 10
    save 60 10000
    stop-writes-on-bgsave-error yes
    rdbcompression yes
    rdbchecksum yes
    dbfilename dump.rdb
    rdb-del-sync-files no
    dir /data
    replica-serve-stale-data yes
    replica-read-only yes
    repl-diskless-sync no
    repl-diskless-sync-delay 5
    repl-diskless-load disabled
    repl-disable-tcp-nodelay no
    replica-priority 100
    acllog-max-len 128
```

c) 创建deployment资源清单

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-single
  labels:
    app: redis-single
spec:
  replicas: 1                  # 副本数  redis 是有状态的服务，单纯增加副本数没用
  selector:                    # 关联pod
    matchLabels:
      app: redis-single
  template:
    metadata:
      labels:
        app: redis-single          # 这里绑定了控制器
    spec:
      containers:
      - name: redis-single
        image: redis:6.0.15
        command: 
        - sh
        - -c
        - redis-server "/mnt/redis.conf"
        ports:
        - containerPort: 6379     # 暴露端口
          protocol: TCP
        resources:
          limits:
            cpu: 100m
            memory: 339Mi
          requests:
            cpu: 10m
            memory: 10Mi
        readinessProbe:             # 探针
          failureThreshold: 2
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          tcpSocket:
            port: 6379
          timeoutSeconds: 5
        volumeMounts:
        - name: redis-config
          mountPath: "/mnt"       # 挂载配置文件到指定目录
          readOnly: true
      volumes:                    # 通过configMap分离配置内容
      - name: redis-config
        configMap:
          name: redis-single-conf
      tolerations:
      - effect: NoExecute
        key: "node.kubernetes.io/not-ready"
        operator: "Exists"
        tolerationSeconds: 30      # 当pod部署的node匹配到此污点后，等待30秒后将Pod转移（故障转移）
      - key: "node.kubernetes.io/unreachable"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 30
```

d) 创建service资源清单

```yaml

```





## 部署Redis集群

​	由于Redis集群的部署比较复杂，所以使用[operator](https://github.com/operator-framework/awesome-operators)来简化部署，并且简化了后期的扩容。

​	operator工具是由coreos开源



a) 部署redis operator

​	选择国内公司写的 [redis operator](https://github.com/ucloud/redis-cluster-operator)，按照官方文档部署



b) 部署redis集群

​	按照官方文档部署





## 部署RabbitMQ集群





## 部署Helm

a) 安装Helm

b) 添加Bitnami仓库

​	[官方文档](https://docs.bitnami.com/tutorials/deploy-apache-airflow-kubernetes-helm/)

```sh
# 添加仓库
$ helm repo add bitnami https://charts.bitnami.com/bitnami

# 查看结果
$ helm repo list

NAME         	URL                                       
ingress-nginx	https://kubernetes.github.io/ingress-nginx
bitnami      	https://charts.bitnami.com/bitnami
```

c) 学习helm

```sh
# 查询插件
$ helm search repo zookeeper 

# 创建插件
$ helm create test

# 插件目录结构
$ cd test
$ tree
.
├── charts          # 依赖文件
├── Chart.yaml      # 版本信息
├── templates       # 配置模板，模板可以读取到 values.yaml 中定义的值
│   ├── deployment.yaml 
│   ├── _helpers.tpl    # 命名模板，存放通用模板段落或者参数
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt      # 部署chart后的信息提示说明
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests          # 如果部署的是web，可以做个连接看是否部署正常
│       └── test-connection.yaml
└── values.yaml      # 配置全局变量或者一些参数


```





## 部署EFK





FIS需求数量：4件

FIS运维数量：4件

主要需求描述：
1、报表模块功能迭代（配置化、动态渲染、界面）：完成度：100%，工时：10个工作日

2、讨论科技信息化部门建设方案 4个工作日

