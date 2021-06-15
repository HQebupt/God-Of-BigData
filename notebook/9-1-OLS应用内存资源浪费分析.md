## 0.集群概况
新的OLS集群规模
- 机器数:18台，256G内存，CPU 55核数
- 内存：4.15 TB
- APP：正在迁移
- 资源分配单位：128M


OLS集群规模
- 机器数：60台机器, 48G内存，CPU 24核数 
- 内存：2.8 T
- APP: 137 app, 占用2.5T
- 资源分配单位：1024M


CPU详细信息：2个物理的，型号-Intel(R) Xeon(R) CPU E5-2430 v2 @ 2.50GHz

|Num | Total内存|总占用| OLS|Spark Stream| Samza|Mem Reserved|
|---|---|---|---|---|---|---|---|
|60 | 2.58T|2.53T|2.1T|229.3G|30G|180G|

- 统计的OLS所有运行的Container最大占用内存1.3T, 存在800G的优化空间。
- 如果提高用户的内存利用率到80%，可以节约出475G的内存。
- Yarn集群，一个Container最大可申请的内存是21G

---
## 1 OLS应用内存资源浪费分析
1. 用户原因：用户申请的内存资源不合理（主要原因）。--- 解决办法：不在增加内存。
2. 计算框架原因：框架的设计机制会造成一定程度的Memory浪费(次要原因)。 ---解决：128M为最小的分配单位，细粒度分配资源。



下面分析一下框架对资源的申请和分配原理，了解为什么OLS实时集群的资源被浪费了。
#### 1.1 OLS框架
OLS对用户申请的内存JavaArgs参数`Xmx1024m`,会加大1.25倍，变成`Xmx1280m`。OLS的AppMaster会以`Xmx1280m`去向Resource Manager申请资源。

OLS增大内存的具体方法见：
`com.sina.ols.yarn.tools.MemUtils#extractMemoryParam(int, java.util.List<java.lang.String>)`

新版本：0.2.3.0 会去掉增大内存的设置，改成用户申请多少，就是多少内存。

### 1.2 RM框架
在YARN中，资源管理由ResourceManager和NodeManager共同完成，其中，ResourceManager中的调度器负责资源的分配，而NodeManager则负责资源的供给和隔离。

AppMaster 向RM申请Container的资源。为什么申请`Xmx1280m`, 最后RM告知Node Manager的Container的资源是2048m呢？

AppMaster 向RM申请资源，RM将请求给公平调度器`FairScheduler`,调度器根据下面三个参数归一化给Container的内存资源。

- maximumAllocation: RM允许最大AM申请Container资源 （集群设置为21G）
- minimumAllocation: 最小分配资源 （默认1024m）
- incrAllocation: 增加Container资源的最小单位 （默认1024m）

Yarn的ResourceManger（简称RM）通过逻辑上的队列分配内存，CPU等资源给application，
- 默认情况下RM允许最大AM申请Container资源为8192MB(“yarn.scheduler.maximum-allocation-mb“)，
- 默认情况下的最小分配资源为1024M(“yarn.scheduler.minimum-allocation-mb“)，AM只能以增量（”yarn.scheduler.minimum-allocation-mb“）和不会超过(“yarn.scheduler.maximum-allocation-mb“)的值去向RM申请资源，
- AM负责将(“mapreduce.map.memory.mb“)和(“mapreduce.reduce.memory.mb“)的值规整到能被(“yarn.scheduler.minimum-allocation-mb“)整除，RM会拒绝申请内存超过8192MB和不能被1024MB整除的资源请求。

> 最后Container 启动的Java参数是以`Xmx1280m`。不是用户设置的`Xmx1024m`,也不是RM给Container分配的`2048m`。这是框架资源浪费的直接原因。

**内存规整的算法**

规整化因子是用来规整化应用程序资源的，应用程序申请的资源如果不是该因子的整数倍，则将被修改为最小的整数倍对应的值，公式为ceil(a/b)*b，其中a是应用程序申请的资源，b为规整化因子。

- FIFO和Capacity Scheduler，规整化因子等于最小可申请资源量，不可单独配置。
- Fair Scheduler：规整化因子通过参数yarn.scheduler.increment-allocation-mb和yarn.scheduler.increment-allocation-vcores设置，默认是1024和1。
- 见源码：`org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler#allocate` ---> `org.apache.hadoop.yarn.util.resource.DominantResourceCalculator#normalize`

**注意**

默认情况下，YARN采用了线程监控的方法判断任务是否超量使用内存，一旦发现超量，则直接将其杀死。由于Cgroups对内存的控制缺乏灵活性（即任务任何时刻不能超过内存上限，如果超过，则直接将其杀死或者报OOM），而Java进程在创建瞬间内存将翻倍，之后骤降到正常值，这种情况下，采用线程监控的方式更加灵活（当发现进程树内存瞬间翻倍超过设定值时，可认为是正常现象，不会将任务杀死），因此YARN未提供Cgroups内存隔离机制。（这可能是OLS加大用户申请内存的原因。）

#### 简述YARN中内存和CPU资源隔离机制

- CPU的资源隔离方案采用了Linux Kernel提供的轻量级资源隔离技术Cgroup。
- 内存资源：**进程监控方案**和**基于轻量级资源隔离技术Cgroups**的方案。默认情况下，YARN采用了进程监控的方案控制内存使用。

##### 进程监控的方案

首先可计算当前每个运行的Container使用的内存总量。但需要注意的是，不能仅凭该内存量是否超过设定的内存最高值来决定是否杀死一个Container。**在创建一个子进程时，JVM采用了"fork()+exec()"模型，这意味着进程创建之后、执行之前会复制一份父进程内存空间，进而使得进程树在某一小段时间内存使用量翻倍。**

为了避免误杀Container，Hadoop赋予每个进程“年龄”属性，并规定刚启动进程的年龄是1，且MonitoringThread线程每更新一次，各个进程年龄加一，在此基础上，选择被杀死Container的标准如下：

- 如果一个Container对应的进程树中所有进程（年龄大于0）总内存超过（用户设置的）最大值的两倍。

- 或者所有年龄大于1的进程总内存量超过（用户设置的）最大值。

  则认为该Container过量使用内存，则向Container发送ContainerEventType.KILL_CONTAINER事件将其杀死。

##### Cgroups

Cgroup会严格限制应用程序的内存使用上限，一旦使用量超过预先定义的上限值，就会将该应用程序“杀死”，因此无法有效地使用Cgroup进行内存资源隔离。



---
### 2 方法1: 解析OLS的workflow.xml文件的Java参数
OLS的应用提交之后，会在HDFS中存储workflow.xml文件。
- workflow.xml 位置：`/user/$user/onlinestream/$ApplicationID`

统计ols实时任务的资源统计，从网页页面抓取基本信息，从HDFS获取 workflow.xml 文件，解析文件的Java args参数，统计出结果。
##### 1.原始日志 ols-stat
```
application_1442477612515_6211  yinquan kafkatoes       OLS_Yarn        root.ad_bp.yinquan      Tue Sep 06 2016 10:37:55 GMT+0800 (CST) N/A     RUNNING UNDEFINED
application_1442477612515_6210  suda    ols_suda_tianyi_cre_news_test   OLS_Yarn        root.suda.suda  Tue Sep 06 2016 10:22:50 GMT+0800 (CST) N/A     RUNNING UNDEFINED
```

##### 2.预处理
```bash

grep "OLS_Yarn" ols-stat | awk '{print "hadoop fs -ls /user/"$2"/onlinestream/"$1}' > jobs
cat jobs | awk -F ' ' '{print $NF}' > jobs_bak

//对jobs_bak vi处理,行尾都添加"/*.xml"字符串：%s/$/String
:%s/$/\/*.xml

for line in `cat jobs_bak`; do directory=`echo $line | awk -F'/' '{print $3"_"$5}'`;mkdir $directory;hadoop fs -get $line ./$directory; rm -f ./$directory/conf.xml;done

```

##### 3.java 分析 workflow 文件
核心的字段数据
```xml
<Parallelism>3</Parallelism>
<JavaArgv>-Xms1024m -Xmx1024m -XX:+UseParallelGC -XX:-UseGCOverheadLimit -XX:ParallelGCThreads=4</JavaArgvv>
<Parallelism>10</Parallelism>
```
- 数据文件：/Users/huangqiang/Documents/sina-ws/Docs/workflow
- 代码文件：/Users/huangqiang/Documents/sina-ws/kafka-eager-eyes/eager-eyes/ols_kafka-0.8.0/src/main/java/demo/OlsStat.java


##### 4.优缺点
1. 速度快，方便操作
2. 统计的信息不准确, 有以下原因：
 - workflow.xml 文件的`xmx`，在提交给Yarn之后，Yarn会额外给App的Container增加内存
 - Container 在运行过程中，如果遇到内存不足时, 发生OOM, Yarn会给它新加1024M内存，重新启动Container。

---
### 方法2: 解析NodeManager的日志
NodeManager日志有Container运行的内存信息，解析这些日志，拿到每个Container一段时间内的最大使用值。合并到ApplicationID中，得到关于Application和Container内存使用的两张表。

##### 1.步骤
1. put 两天的NodeManager日志到HDFS: `/user/huangqiang/ols-stat/`
2. 解析一份user2applicationId的列表, put 到HDFS: `/user/huangqiang/ols-new-info`
3. Java程序：解析日志，用HashMap解决。


##### 2 NodeManager的日志格式
2.1 过滤好所需要的日志，需要的是28个字段的一行，12、15字段是内存信息。
```
2016-09-05 00:00:01,587 INFO org.apache.hadoop.yarn.server.nodemanager.containermanager.monitor.ContainersMonitorImpl: Memory usage of ProcessTree 155585 for container-id container_1442477612515_5900_01_000007: 218.4 MB of 2 GB physical memory used; 2.0 GB of 4.2 GB virtual memory used
2016-09-05 00:00:01,668 INFO org.apache.hadoop.yarn.server.nodemanager.containermanager.monitor.ContainersMonitorImpl: Memory usage of ProcessTree 2917 for container-id container_1442477612515_3787_02_000007: 806.0 MB of 3 GB physical memory used; 3.8 GB of 6.3 GB virtual memory used

```


2.2 put 两天的NodeManager日志到HDFS: 预先建立HDFS路径：`hadoop fs -mkdir /user/huangqiang/ols-stat/` 



**ols-put2hdfs.sh**
```bash
. /etc/bashrc
. /etc/profile
set -x

# /data0/hadoop/log/yarn/yarn-mapred-nodemanager-yz4854.hadoop.data.sina.com.cn.log.2016-08-07
file905=/data0/hadoop/log/yarn/yarn-mapred-nodemanager-`hostname`.log.2016-09-05
file=/data0/hadoop/log/yarn/yarn-mapred-nodemanager-`hostname`.log.2016-09-06
hadoop fs -copyFromLocal $file905 /user/huangqiang/ols-stat/
hadoop fs -copyFromLocal $file /user/huangqiang/ols-stat/

```

2.3 通道机多命令执行，put数据到hdfs
```bash
### put to hdfs
export RSYNC_PASSWORD=qwe123 &&  rsync ols-put2hdfs.sh root@10.39.3.75::shellResult/huangqiang/ols-stat/
export RSYNC_PASSWORD=qwe123 && rsync root@10.39.3.75::shellResult/huangqiang/ols-stat/ols-put2hdfs.sh ./ && sh ols-put2hdfs.sh
```
##### 3 user对应的Application的信息
从OLS集群的页面复制下来，经过awk处理成如下的信息 ols-new-info，put到HDFS：`/user/huangqiang/ols-new-info`
```
application_1442477612515_6206 sinadata
application_1442477612515_6197 yinquan
application_1442477612515_6194 dw
```

##### 4 Java 读取HDFS文件
- 代码工程：sina-ws/kafka-eager-eyes/eager-eyes/ols_kafka-0.8.0
- 运行机器: 3.142
- cmd: ` nohup /usr/local/jdk1.7.0_67/bin/java -cp ols-memory-statics.jar demo.OlsMemoryNew > ols-memory-statics.log &`

将`core-site.xml` `hdfs.xml` 放到maven工程的resource目录下，主函数读取HDFS文件的java关键代码如下：
```java
        Configuration conf = new Configuration();
        FileSystem fs = FileSystem.get(conf);
        Path file = new Path("/user/huangqiang/ols-new-info");
        FSDataInputStream fsin = fs.open(file);
        BufferedReader br = new BufferedReader(new InputStreamReader(fsin));
        while ((line = br.readLine()) != null) {
            // do something
        }
        
        br.close();
        fsin.close();
        fs.close();
```

##### 5.1 程序结果：Application的内存使用情况
- 60台机器，数据文件共 15.2 G，单线程运行时间：14 min。
- 考虑是否可以改成多线程版本 or Mapreduce，提升运行效率。

|user | appId | used | mem | total mem | 可优化的mem (MB)|
|---  |---    |---   |---  |---    |---  |
|sina_recmd | application_1442477612515_5461 | 18236.8 | 123904 | 14.72 | 105667.2|
|sinadata | application_1442477612515_4655 | 58903.4 | 120832 | 48.75 | 61928.6|
|dw | application_1442477612515_6195 | 151756.8 | 199680 | 76.00 | 47923.2|

##### 5.2 Application对应的Container的内存使用情况

|containerId|used mem| total mem(MB) | rate(%)|
|---|---|---|----|
|container_1442477612515_0038_03_000001|443.9|1024.0|43.34 |
|container_1442477612515_0038_03_000002|594.7|2048.0|29.03 |
|container_1442477612515_0038_03_000003|644.0|2048.0|31.44 |