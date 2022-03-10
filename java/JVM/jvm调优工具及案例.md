**前置启动程序** 

事先启动一个web应用程序，用jps查看其进程id，接着用各种jdk自带命令优化应用 

## Jmap

**此命令可以用来查看内存信息，实例个数以及占用内存大小** 

![image-20220309171150394](assets/image-20220309171150394.png)

打开log.txt，文件内容如下：

![image-20220309171203281](assets/image-20220309171203281.png)

- num：序号 
- instances：实例数量 
- bytes：占用空间大小 
- class name：类名称:`[C is a char[]，[S is a short[]，[I is a int[]，[B is a byte[]，[[I is a int[][]`

**堆信息**

![image-20220309171541329](assets/image-20220309171541329.png)

**堆内存dump**：当前程序内存的快照

```sh
jmap ‐dump:format=b,file=eureka.hprof 14660
```

![image-20220309172142096](assets/image-20220309172142096.png)

也可以设置内存溢出自动导出dump文件(内存很大的时候，可能会导不出来) 

1. -XX:+HeapDumpOnOutOfMemoryError 
2. -XX:HeapDumpPath=./ （路径） 

```java
public class OOMTest {
   public static List<Object> list = new ArrayList<>();

   // JVM设置    
   // -Xms10M -Xmx10M -XX:+PrintGCDetails -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=D:\jvm.dump 
   public static void main(String[] args) {
      List<Object> list = new ArrayList<>();
      int i = 0;
      int j = 0;
      while (true) {
         list.add(new User(i++, UUID.randomUUID().toString()));
         new User(j--, UUID.randomUUID().toString());
      }
   }
}
```

**可以用jvisualvm命令工具导入该dump文件分析** 

![image-20220309172506952](assets/image-20220309172506952.png)

## Jstack

用jstack加进程id查找死锁，见如下示例

```java
public class DeadLockTest {

   private static Object lock1 = new Object();
   private static Object lock2 = new Object();

   public static void main(String[] args) {
      new Thread(() -> {
         synchronized (lock1) {
            try {
               System.out.println("thread1 begin");
               Thread.sleep(5000);
            } catch (InterruptedException e) {
            }
            synchronized (lock2) {
               System.out.println("thread1 end");
            }
         }
      }).start();

      new Thread(() -> {
         synchronized (lock2) {
            try {
               System.out.println("thread2 begin");
               Thread.sleep(5000);
            } catch (InterruptedException e) {
            }
            synchronized (lock1) {
               System.out.println("thread2 end");
            }
         }
      }).start();

      System.out.println("main thread end");
   }
}
```

![image-20220309172838438](assets/image-20220309172838438.png)

- "Thread-1" 线程名 
- prio=5 优先级=5
- tid=0x000000001fa9e000 线程id
- nid=0x2d64 线程对应的本地线程标识nid
- **java.lang.Thread.State**: BLOCKED 线程状态

![image-20220309172954903](assets/image-20220309172954903.png)

还可以用jvisualvm自动检测死锁 

![image-20220309173019862](assets/image-20220309173019862.png)

**远程连接jvisualvm**

**启动普通的jar程序JMX端口配置：**

```sh
# -Dcom.sun.management.jmxremote.port 为远程机器的JMX端口
# -Djava.rmi.server.hostname 为远程机器IP
java -Dcom.sun.management.jmxremote.port=8888 -Djava.rmi.server.hostname=192.168.65.60 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -jar microservice-eureka-server.jar
```

**tomcat的JMX配置：在catalina.sh文件里的最后一个JAVA_OPTS的赋值语句下一行增加如下配置行** 

```txt
JAVA_OPTS="$JAVA_OPTS ‐Dcom.sun.management.jmxremote.port=8888 ‐Djava.rmi.server.hostname=192.168.50.60 ‐Dcom.sun.ma nagement.jmxremote.ssl=false ‐Dcom.sun.management.jmxremote.authenticate=false"
```

## Jinfo

查看正在运行的Java应用程序的扩展参数 

查看jvm的参数 

![image-20220309173630248](assets/image-20220309173630248.png)

查看java系统参数 

![image-20220309173638159](assets/image-20220309173638159.png)

## Jstat

jstat命令可以查看堆内存各部分的使用量，以及加载类的数量。命令的格式如下： 

jstat [-命令选项] [vmid] [间隔时间(毫秒)] [查询次数] 

注意：使用的jdk版本是jdk8 

**垃圾回收统计** 

**jstat -gc pid 最常用**，可以评估程序内存使用及GC压力整体情况

![image-20220309173720257](assets/image-20220309173720257.png)

- S0C：第一个幸存区的大小，单位KB
- S1C：第二个幸存区的大小
- S0U：第一个幸存区的使用大小
- S1U：第二个幸存区的使用大小
- EC：伊甸园区的大小
- EU：伊甸园区的使用大小
- OC：老年代大小
- OU：老年代使用大小
- MC：方法区大小(元空间)
- MU：方法区使用大小
- CCSC:压缩类空间大小
- CCSU:压缩类空间使用大小
- YGC：年轻代垃圾回收次数
- YGCT：年轻代垃圾回收消耗时间，单位s
- FGC：老年代垃圾回收次数 
- FGCT：老年代垃圾回收消耗时间，单位s
- GCT：垃圾回收消耗总时间，单位s

**堆内存统计**

![image-20220309173804170](assets/image-20220309173804170.png)

- NGCMN：新生代最小容量
- NGCMX：新生代最大容量
- NGC：当前新生代容量
- S0C：第一个幸存区大小
- S1C：第二个幸存区的大小
- EC：伊甸园区的大小
- OGCMN：老年代最小容量
- OGCMX：老年代最大容量
- OGC：当前老年代大小
- OC:当前老年代大小
- MCMN:最小元数据容量
- MCMX：最大元数据容量
- MC：当前元数据空间大小
- CCSMN：最小压缩类空间大小
- CCSMX：最大压缩类空间大小
- CCSC：当前压缩类空间大小
- YGC：年轻代gc次数
- FGC：老年代GC次数

**新生代垃圾回收统计**

![image-20220309173823844](assets/image-20220309173823844.png)

- S0C：第一个幸存区的大小
- S1C：第二个幸存区的大小
- S0U：第一个幸存区的使用大小
- S1U：第二个幸存区的使用大小
- TT:对象在新生代存活的次数
- MTT:对象在新生代存活的最大次数
- DSS:期望的幸存区大小
- EC：伊甸园区的大小
- EU：伊甸园区的使用大小
- YGC：年轻代垃圾回收次数
- YGCT：年轻代垃圾回收消耗时间

**新生代内存统计**

![image-20220309173842500](assets/image-20220309173842500.png)

- NGCMN：新生代最小容量
- NGCMX：新生代最大容量
- NGC：当前新生代容量
- S0CMX：最大幸存1区大小
- S0C：当前幸存1区大小
- S1CMX：最大幸存2区大小
- S1C：当前幸存2区大小
- ECMX：最大伊甸园区大小
- EC：当前伊甸园区大小
- YGC：年轻代垃圾回收次数
- FGC：老年代回收次数

**老年代垃圾回收统计**

![image-20220309173910029](assets/image-20220309173910029.png)

- MC：方法区大小
- MU：方法区使用大小
- CCSC:压缩类空间大小
- CCSU:压缩类空间使用大小
- OC：老年代大小
- OU：老年代使用大小
- YGC：年轻代垃圾回收次数
- FGC：老年代垃圾回收次数
- FGCT：老年代垃圾回收消耗时间
- GCT：垃圾回收消耗总时间

**老年代内存统计**

![image-20220309173921651](assets/image-20220309173921651.png)

- OGCMN：老年代最小容量
- OGCMX：老年代最大容量
- OGC：当前老年代大小
- OC：老年代大小
- YGC：年轻代垃圾回收次数
- FGC：老年代垃圾回收次数
- FGCT：老年代垃圾回收消耗时间
- GCT：垃圾回收消耗总时间

**元数据空间统计**

![image-20220309173946352](assets/image-20220309173946352.png)

- MCMN:最小元数据容量
- MCMX：最大元数据容量
- MC：当前元数据空间大小 
- CCSMN：最小压缩类空间大小
- CCSMX：最大压缩类空间大小
- CCSC：当前压缩类空间大小
- YGC：年轻代垃圾回收次数
- FGC：老年代垃圾回收次数
- FGCT：老年代垃圾回收消耗时间
- GCT：垃圾回收消耗总时间

![image-20220309173959383](assets/image-20220309173959383.png)

- S0：幸存1区当前使用比例
- S1：幸存2区当前使用比例
- E：伊甸园区使用比例
- O：老年代使用比例
- M：元数据区使用比例
- CCS：压缩使用比例
- YGC：年轻代垃圾回收次数
- FGC：老年代垃圾回收次数
- FGCT：老年代垃圾回收消耗时间
- GCT：垃圾回收消耗总时间

**JVM运行情况预估**

用 jstat gc -pid 命令可以计算出如下一些关键数据，有了这些数据就可以采用之前介绍过的优化思路，先给自己的系统设置一些初始性的JVM参数，比如堆内存大小，年轻代大小，Eden和Survivor的比例，老年代的大小，大对象的阈值，大龄对象进入老年代的阈值等。

- **年轻代对象增长的速率**

  可以执行命令 jstat -gc pid 1000 10 (每隔1秒执行1次命令，共执行10次)，通过观察EU(eden区的使用)来估算每秒eden大概新增多少对象，如果系统负载不高，可以把频率1秒换成1分钟，甚至10分钟来观察整体情况。注意，一般系统可能有高峰期和日常期，所以需要在不同的时间分别估算不同情况下对象增长速率。

- **Young GC的触发频率和每次耗时**

  知道年轻代对象增长速率我们就能推根据eden区的大小推算出Young GC大概多久触发一次，Young GC的平均耗时可以通过 YGCT/YGC 公式算出，根据结果我们大概就能知道**系统大概多久会因为Young GC的执行而卡顿多久。**

- **每次Young GC后有多少对象存活和进入老年代**

  这个因为之前已经大概知道Young GC的频率，假设是每5分钟一次，那么可以执行命令 jstat -gc pid 300000 10 ，观察每次结果eden，survivor和老年代使用的变化情况，在每次gc后eden区使用一般会大幅减少，survivor和老年代都有可能增长，这些增长的对象就是每次Young GC后存活的对象，同时还可以看出每次Young GC后进去老年代大概多少对象，从而可以推算出**老年代对象增长速率。**

- **Full GC的触发频率和每次耗时**

  知道了老年代对象的增长速率就可以推算出Full GC的触发频率了，Full GC的每次耗时可以用公式 FGCT/FGC 计算得出。

- **优化思路**其实简单来说就是尽量让每次Young GC后的存活对象小于Survivor区域的50%，都留存在年轻代里。尽量别让对象进入老年代。尽量减少Full GC的频率，避免频繁Full GC对JVM性能的影响。



## 阿里巴巴 Arthas

**Arthas** 是 Alibaba 在 2018 年 9 月开源的 **Java 诊断**工具。支持 JDK6+， 采用命令行交互模式，可以方便的定位和诊断线上程序运行问题。**Arthas** 官方文档十分详细，详见：[*https://alibaba.github.io/arthas*](https://alibaba.github.io/arthas)

### **Arthas使用场景**

得益于 **Arthas** 强大且丰富的功能，让 **Arthas** 能做的事情超乎想象。下面仅仅列举几项常见的使用情况，更多的使用场景可以在熟悉了 **Arthas** 之后自行探索。

1. 是否有一个全局视角来查看系统的运行状况？
2. 为什么 CPU 又升高了，到底是哪里占用了 CPU ？
3. 运行的多线程有死锁吗？有阻塞吗？
4. 程序运行耗时很长，是哪里耗时比较长呢？如何监测呢？
5. 这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
6. 我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？
7. 遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？
8. 有什么办法可以监控到 JVM 的实时运行状态？

### **Arthas使用**

```sh
# github下载arthas
wget https://alibaba.github.io/arthas/arthas-boot.jar
# 或者 Gitee 下载
wget https://arthas.gitee.io/arthas-boot.jar
```

用java -jar运行即可，可以识别机器上所有Java进程(我们这里之前已经运行了一个Arthas测试程序，代码见下方)

![image-20220310152802751](assets/image-20220310152802751.png)

```java
package com.tuling.jvm;

import java.util.HashSet;

public class Arthas {

    private static HashSet hashSet = new HashSet();

    public static void main(String[] args) {
        // 模拟 CPU 过高
        cpuHigh();
        // 模拟线程死锁
        deadThread();
        // 不断的向 hashSet 集合增加数据
        addHashSetThread();
    }

    /**
     * 不断的向 hashSet 集合添加数据
     */
    public static void addHashSetThread() {
        // 初始化常量
        new Thread(() -> {
            int count = 0;
            while (true) {
                try {
                    hashSet.add("count" + count);
                    Thread.sleep(1000);
                    count++;
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }

    public static void cpuHigh() {
        new Thread(() -> {
            while (true) {

            }
        }).start();
    }

    /**
     * 死锁
     */
    private static void deadThread() {
        /** 创建资源 */
        Object resourceA = new Object();
        Object resourceB = new Object();
        // 创建线程
        Thread threadA = new Thread(() -> {
            synchronized (resourceA) {
                System.out.println(Thread.currentThread() + " get ResourceA");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread() + "waiting get resourceB");
                synchronized (resourceB) {
                    System.out.println(Thread.currentThread() + " get resourceB");
                }
            }
        });

        Thread threadB = new Thread(() -> {
            synchronized (resourceB) {
                System.out.println(Thread.currentThread() + " get ResourceB");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread() + "waiting get resourceA");
                synchronized (resourceA) {
                    System.out.println(Thread.currentThread() + " get resourceA");
                }
            }
        });
        threadA.start();
        threadB.start();
    }
}
```

选择进程序号1，进入进程信息操作

![image-20220310152945716](assets/image-20220310152945716.png)

输入**dashboard**可以查看整个进程的运行情况，线程、内存、GC、运行环境信息：

![image-20220310153411412](assets/image-20220310153411412.png)

![image-20220310153415472](assets/image-20220310153415472.png)

输入 **thread加上线程ID** 可以查看线程堆栈

![image-20220310153425189](assets/image-20220310153425189.png)

输入 **thread -b** 可以查看线程死锁 

![image-20220310153433100](assets/image-20220310153433100.png)

输入 **jad加类的全名** 可以反编译，这样可以方便我们查看线上代码是否是正确的版本

![image-20220310153728522](assets/image-20220310153728522.png)

使用 **ognl** 命令可以查看线上系统变量的值，甚至可以修改变量的值 

![image-20220310153802726](assets/image-20220310153802726.png)

更多命令使用可以用help命令查看，或查看文档：https://alibaba.github.io/arthas/commands.html#arthas 



## **GC日志详解** 

对于java应用我们可以通过一些配置把程序运行过程中的gc日志全部打印出来，然后分析gc日志得到关键性指标，分析GC原因，调优JVM参数。

打印GC日志方法，在JVM参数里增加参数，%t 代表时间

```sh
-Xloggc:./gc-%t.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps  -XX:+PrintGCTimeStamps -XX:+PrintGCCause  
-XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M
```

Tomcat则直接加在JAVA_OPTS变量里。

### **如何分析GC日志**

运行程序加上对应gc日志

```sh
java -jar -Xloggc:./gc-%t.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps  -XX:+PrintGCTimeStamps -XX:+PrintGCCause  
-XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M microservice-eureka-server.jar
```

下图中是我截取的JVM刚启动的一部分GC日志 

![image-20220310155022965](assets/image-20220310155022965.png)

我们可以看到图中第一行红框，是项目的配置参数。这里不仅配置了打印GC日志，还有相关的VM内存参数

第二行红框中的是在这个GC时间点发生GC之后相关GC情况

1. 对于**2.909：**  这是从jvm启动开始计算到这次GC经过的时间，前面还有具体的发生时间日期。
2. Full GC(Metadata GC Threshold)指这是一次full gc，括号里是gc的原因， PSYoungGen是年轻代的GC，ParOldGen是老年代的GC，Metaspace是元空间的GC
3. 6160K->0K(141824K)，这三个数字分别对应GC之前占用年轻代的大小，GC之后年轻代占用，以及整个年轻代的大小
4. 112K->6056K(95744K)，这三个数字分别对应GC之前占用老年代的大小，GC之后老年代占用，以及整个老年代的大小
5. 6272K->6056K(237568K)，这三个数字分别对应GC之前占用堆内存的大小，GC之后堆内存占用，以及整个堆内存的大小
6. 20516K->20516K(1069056K)，这三个数字分别对应GC之前占用元空间内存的大小，GC之后元空间内存占用，以及整个元空间内存的大小
7. 0.0209707是该时间点GC总耗费时间。 

从日志可以发现几次fullgc都是由于元空间不够导致的，所以我们可以将元空间调大点

```sh
java -jar -Xloggc:./gc-adjust-%t.log -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M -XX:+PrintGCDetails -XX:+PrintGCDateStamps  
-XX:+PrintGCTimeStamps -XX:+PrintGCCause  -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M 
microservice-eureka-server.jar
```

调整完我们再看下gc日志发现已经没有因为元空间不够导致的fullgc了

对于CMS和G1收集器的日志会有一点不一样，也可以试着打印下对应的gc日志分析下，可以发现gc日志里面的gc步骤跟我们之前讲过的步骤是类似的

```java
public class HeapTest {

    byte[] a = new byte[1024 * 100];  //100KB

    public static void main(String[] args) throws InterruptedException {
        ArrayList<HeapTest> heapTests = new ArrayList<>();
        while (true) {
            heapTests.add(new HeapTest());
            Thread.sleep(10);
        }
    }
}
```

### **CMS**

```sh
-Xloggc:d:/gc-cms-%t.log -Xms50M -Xmx50M -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M -XX:+PrintGCDetails -XX:+PrintGCDateStamps  
 -XX:+PrintGCTimeStamps -XX:+PrintGCCause  -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M 
 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC   
```

### **G1**

```sh
-Xloggc:d:/gc-g1-%t.log -Xms50M -Xmx50M -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M -XX:+PrintGCDetails -XX:+PrintGCDateStamps  
 -XX:+PrintGCTimeStamps -XX:+PrintGCCause  -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M -XX:+UseG1GC 
```

上面的这些参数，能够帮我们查看分析GC的垃圾收集情况。但是如果GC日志很多很多，成千上万行。就算你一目十行，看完了，脑子也是一片空白。所以我们可以借助一些功能来帮助我们分析，这里推荐一个gceasy([https://gceasy.io](https://gceasy.io/))，可以上传gc文件，然后他会利用可视化的界面来展现GC情况。具体下图所示 

![image-20220310160011758](assets/image-20220310160011758.png)

上图我们可以看到年轻代，老年代，以及永久代的内存分配，和最大使用情况

![image-20220310160021758](assets/image-20220310160021758.png)

上图我们可以看到堆内存在GC之前和之后的变化，以及其他信息。 

这个工具还提供基于机器学习的JVM智能优化建议，当然现在这个功能需要付费

**JVM参数汇总查看命令** 

java -XX:+PrintFlagsInitial 表示打印出所有参数选项的默认值 

java -XX:+PrintFlagsFinal 表示打印出所有参数选项在运行程序时生效的值 





## 案例

### 案例一： jstack找出占用cpu最高的线程堆栈信息

```java
package com.tuling.jvm;

/**
 * 运行此代码，cpu会飙高
 */
public class Math {

    public static final int initData = 666;
    public static User user = new User();

    public int compute() {  //一个方法对应一块栈帧内存区域
        int a = 1;
        int b = 2;
        int c = (a + b) * 10;
        return c;
    }

    public static void main(String[] args) {
        Math math = new Math();
        while (true){
            math.compute();
        }
    }
}
```

1，使用命令top -p <pid> ，显示你的java进程的内存情况，pid是你的java进程号，比如19663

![image-20220309173501518](assets/image-20220309173501518.png)

2，按H，获取每个线程的内存情况 

![image-20220309173533541](assets/image-20220309173533541.png)

3，找到内存和cpu占用最高的线程tid，比如19664 

4，转为十六进制得到 0x4cd0，此为线程id的十六进制表示

5，执行 jstack 19663|grep -A 10 4cd0，得到线程堆栈信息中 4cd0 这个线程所在行的后面10行，从堆栈中可以发现导致cpu飙高的调 

用方法 

![image-20220309173603108](assets/image-20220309173603108.png)

6，查看对应的堆栈信息找出可能存在问题的代码 

### 案例二：**系统频繁Full GC导致系统卡顿是怎么回事**

- 机器配置：2核4G
- JVM内存大小：2G
- 系统运行时间：7天
- 期间发生的Full GC次数和耗时：500多次，200多秒
- 期间发生的Young GC次数和耗时：1万多次，500多秒

大致算下来每天会发生70多次Full GC，平均每小时3次，每次Full GC在400毫秒左右；

每天会发生1000多次Young GC，每分钟会发生1次，每次Young GC在50毫秒左右。

JVM参数设置如下：

```sh
-Xms1536M -Xmx1536M -Xmn512M -Xss256K -XX:SurvivorRatio=6  -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M 
-XX:+UseParNewGC  -XX:+UseConcMarkSweepGC  -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly 
```

![image-20220309174235388](assets/image-20220309174235388.png)

大家可以结合**对象挪动到老年代那些规则**推理下我们这个程序可能存在的一些问题

经过分析感觉可能会由于对象动态年龄判断机制导致full gc较为频繁

为了给大家看效果，我模拟了一个示例程序(见课程对应工程代码：jvm-full-gc)，打印了jstat的结果如下：

```sh
jstat -gc 13456 2000 10000
```

![image-20220309174256311](assets/image-20220309174256311.png)

对于对象动态年龄判断机制导致的full gc较为频繁可以先试着优化下JVM参数，把年轻代适当调大点：

```sh
-Xms1536M -Xmx1536M -Xmn1024M -Xss256K -XX:SurvivorRatio=6  -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M 
-XX:+UseParNewGC  -XX:+UseConcMarkSweepGC  -XX:CMSInitiatingOccupancyFraction=92 -XX:+UseCMSInitiatingOccupancyOnly 
```

![image-20220309174316403](assets/image-20220309174316403.png)

优化完发现没什么变化，**full gc的次数比minor gc的次数还多了** 

![image-20220309174324665](assets/image-20220309174324665.png)

我们可以推测下full gc比minor gc还多的原因有哪些？

1. 元空间不够导致的多余full gc
2. 显示调用System.gc()造成多余的full gc，这种一般线上尽量通过-XX:+DisableExplicitGC参数禁用，如果加上了这个JVM启动参数，那么代码中调用System.gc()没有任何效果
3. 老年代空间分配担保机制

最快速度分析完这些我们推测的原因以及优化后，我们发现young gc和full gc依然很频繁了，而且看到有大量的对象频繁的被挪动到老年代，这种情况我们可以借助jmap命令大概看下是什么对象

![image-20220309174605121](assets/image-20220309174605121.png)

查到了有大量User对象产生，这个可能是问题所在，但不确定，还必须找到对应的代码确认，如何去找对应的代码了？

1、代码里全文搜索生成User对象的地方(适合只有少数几处地方的情况)

2、如果生成User对象的地方太多，无法定位具体代码，我们可以同时分析下占用cpu较高的线程，一般有大量对象不断产生，对应的方法代码肯定会被频繁调用，占用的cpu必然较高

可以用上面讲过的jstack或jvisualvm来定位cpu使用较高的代码，最终定位到的代码如下：

```java
import java.util.ArrayList;

@RestController
public class IndexController {

    @RequestMapping("/user/process")
    public String processUserData() throws InterruptedException {
        ArrayList<User> users = queryUsers();

        for (User user: users) {
            //TODO 业务处理
            System.out.println("user:" + user.toString());
        }
        return "end";
    }

    /**
     * 模拟批量查询用户场景
     * @return
     */
    private ArrayList<User> queryUsers() {
        ArrayList<User> users = new ArrayList<>();
        for (int i = 0; i < 5000; i++) {
            users.add(new User(i,"zhuge"));
        }
        return users;
    }
}
```

同时，java的代码也是需要优化的，一次查询出500M的对象出来，明显不合适，要根据之前说的各种原则尽量优化到合适的值，尽量消除这种朝生夕死的对象导致的full gc

### 案例三：内存泄露到底是怎么回事

再给大家讲一种情况，一般电商架构可能会使用多级缓存架构，就是redis加上JVM级缓存，大多数同学可能为了图方便对于JVM级缓存就简单使用一个hashmap，于是不断往里面放缓存数据，但是很少考虑这个map的容量问题，结果这个缓存map越来越大，一直占用着老年代的很多空间，时间长了就会导致full gc非常频繁，这就是一种内存泄漏，对于一些老旧数据没有及时清理导致一直占用着宝贵的内存资源，时间长了除了导致full gc，还有可能导致OOM。

这种情况完全可以考虑采用一些成熟的JVM级缓存框架来解决，比如ehcache等自带一些LRU数据淘汰算法的框架来作为JVM级的缓存。

补充：

- **内存溢出（Out Of Memory）** ：就是申请内存时，JVM没有足够的内存空间。通俗说法就是去蹲坑发现坑位满了。
- **内存泄露 （Memory Leak）**：就是申请了内存，但是没有释放，导致内存空间浪费。通俗说法就是有人占着茅坑不拉屎。