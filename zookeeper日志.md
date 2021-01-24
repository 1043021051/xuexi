zookeeper源码分析
----
### Zookeeper简介
Zookeeper是⼀个开源的分布式协调服务，其设计⽬标是将那些复杂的且容易出错的分布式⼀致性服务封装起来，构成⼀个⾼效可靠的原语集，并以⼀些简单的接⼝提供给⽤户使⽤。

zookeeper是⼀个典型的分布式数据⼀致性的解决⽅案，分布式应⽤程序可以基于它实现诸如数据订阅/发布、负载均衡、命名服务、集群管理、分布式锁和分布式队列等功能。

### 一、`zookeeper`日志类型

`zookeeper`中有两类日志,分别是：

*   1、事务日志`log`
*   2、快照日志`snapshot`

`事务日志 :` 顾名思义，就是用于存放事务执行的相关信息，如`zxid`、`cxid`等。至于什么是`zookeeper`事务，放到后面的文章中讲。

`快照日志 `: `zookeeper` 数据节点数据是运行在内存中的，当然内存保存这些结点的数据不可能无限大，而且数据节点的内容是动态变化的，因此`zookeeper`提供一中奖数据节点持久化的机制，每隔一段时间，`zookeeper`会将内存中的数据节点`DataTree`序列到磁盘中，因此就形成了我们的快照日志。

在`zookeeper`源码调试的过程中，诸如服务端的启动日志、客户端的启动日志、请求的相关日志，由于`zookeeper`采用`log4j`进行日志打印，因此`log4j`必须在类路径下查找`log4j.properties`文件，如果启动的过程中找不到该文件，则报错如下：

```
log4j:WARN No appenders could be found for logger (org.apache.log4j.jmx.HierarchyDynamicMBean).
log4j:WARN Please initialize the log4j system properly.
log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
```

因此，必须将`log4j.properties`放到类路径下,所以将`conf`目录下的`log4j.properties`拷贝一份至上图的目录中即可！

### 二、`zookeeper`日志可视化

上面说到`zookeeper`中有两种日志类型，但很遗憾，上面的日志你都无法直接看到，因为都是采用二进制进行保存的。查看`zookeeper`源代码也可以知道，`zookeeper`日志输出体现在`FileTxnLog.java`文件的`append()`方法中，具体如下：

```
// 写日志文件，采用log. + 哈希值的命名形式保存文件
logFileWrite = new File(logDir, ("log." + Long.toHexString(hdr.getZxid())));
// 文件输出流
fos = new FileOutputStream(logFileWrite);
// 采用BufferedOutputStream包裹fos成缓冲输出流
logStream=new BufferedOutputStream(fos);
// 这是重点，这里采用org.apache.jute.BinaryOutputArchive二进制构件对logStream进行包裹，输出二进制数据
oa = BinaryOutputArchive.getArchive(logStream);
```

既然是二进制日志文件，那么我们直接打开该文件肯定是乱码嘛！怎么办呢？下面提供两种方法，这两种方法都是基于`zookeeper`提供的`LogFormatter.java`工具类来实现的。

*   1、在`idea`中开该类，然后运行该类的`main`方法的同时传入你想查看的日志文件路径即可
*   2、采用命令行`java -classpath xxx.jar org.apache.zookeeper.server.LogFormatter logFilePath`的形式进行查看

第一种方式：

这里假设我的日志文件存放路径为`E:\\resources\\zookeeper-3.4.11\\conf\\log\\version-2\\log.1`，当我运行`main`方法时，如下图：

然后就可以在控制台输出日志了，输出样例如下：

```
ZooKeeper Transactional Log File with dbid 0 txnlog format version 2
21-1-23 下午08时39分23秒 session 0x100022b44190000 cxid 0x0 zxid 0x1 createSession 30000

21-1-23 下午08时39分55秒 session 0x100022b44190000 cxid 0x0 zxid 0x2 closeSession null
EOF reached after 2 txns.

```

第二种方式：

*   1、在任意目录下新建一个临时目录,随便命名为`logSee`,将`slf4j-api-1.6.1.jar`和`zookeeper-3.4.11.jar`两个文件放到该目录下，然后打开命令行，执行`java -classpath .;slf4j-api-1.6.1.jar;zookeeper-3.4.11.jar org.apache.zookeeper.server.LogFormatter E:\\resources\\zookeeper-3.4.11\\conf\\log\\version-2\\log.1`

其中两个`jar`包之间采用分号`;`分隔而不是冒号，`org.apache.zookeeper.server.LogFormatter`表示`LogFormatter.java`的全路径命名(`即包全名+类名`),`E:\\resources\\zookeeper-3.4.11\\conf\\log\\version-2\\log.1`表示你想查看的日志文件。


### 三、`zookeeper`日志清理机制

`zookeeper`是将日志以文件的形式存放在磁盘中，久而久之，磁盘的文件就越来越多，`zookeeper`提供了一种定期清理日志和快照文件的机制。

`QuorumPeerMain.java`是`zookeeper`的启动类，在`zookeeper`启动之前会创建一个`DatadirCleanupManager`对象进行数据清理任务，`DatadirCleanupManager`会根据你配置文件的配置决定是否清理以及每隔多久进行清理，其底层原理采用`JDK`的`Timer`定时器实现，下面将对其实现机制从源码的角度进行分析：

先看`QuorumPeerMain`类，在其`initializeAndRun()`方法执行时会创建一个`DatadirCleanupManager`对象，并将`zoo.cfg`配置文件的相关配置传递给该对象，源码如下：

```
protected void initializeAndRun(String[] args)
        throws ConfigException, IOException
    {
        QuorumPeerConfig config = new QuorumPeerConfig();
        if (args.length == 1) {
            config.parse(args[0]);
        }

        // 启动数据目录清理定时任务，传递的参数有数据目录DataDir，日志目录LogDir,以及快照保留数量count，默认>3,最后一个是每个多长时间进行清理
        DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
                .getDataDir(), config.getDataLogDir(), config
                .getSnapRetainCount(), config.getPurgeInterval());
        purgeMgr.start();

        if (args.length == 1 && config.servers.size() > 0) {
            runFromConfig(config);
        } else {
            LOG.warn("Either no config or no quorum defined in config, running "
                    + " in standalone mode");
            // there is only server in the quorum -- run as standalone
            ZooKeeperServerMain.main(args);
        }
    }

```

你可以在配置文件`zoo.cfg`文件中增加两个配置，分别是`autopurge.snapRetainCount`和`autopurge.snapRetainCount` 。`autopurge.purgeInterval`就是设置多少小时清理一次。而`autopurge.snapRetainCount`是设置保留多少个`快照文件snapshot`，之前的多有都删除。

有一点性能问题就是，一般`zookeeper`是运行在集群中，业务会比较繁忙，如果每隔多久去清理势必会影响性能，我们会想能否有一种在集群不繁忙的时候去执行清理操作，比如在每晚的12点。但是很遗憾，`zookeeper`并没有提供相应的实现，`zookeeper`采用`Timer`的方式去实现，而不是像`quartz`,`Spring`一样提供`cron`表达式配置。但注意，并不是说`Timer`不能实现指定时间执行，而是说`zookeeper`没有实现而已。因为`quartz`,`Spring`底层还是使用`Timer`和`Executors`去实现的嘛！

再来看看`DatadirCleanupManager`的`start()`方法：

```
 public void start() {
        if (PurgeTaskStatus.STARTED == purgeTaskStatus) {
            LOG.warn("Purge task is already running.");
            return;
        }
        // Don't schedule the purge task with zero or negative purge interval.
        if (purgeInterval <= 0) {
            LOG.info("Purge task is not scheduled.");
            return;
        }
        // 创建TIMER
        timer = new Timer("PurgeTask", true);
        // 创建定时任务
        TimerTask task = new PurgeTask(dataLogDir, snapDir, snapRetainCount);
        // 每隔多少小时执行
        timer.scheduleAtFixedRate(task, 0, TimeUnit.HOURS.toMillis(purgeInterval));
        purgeTaskStatus = PurgeTaskStatus.STARTED;
    }
```

