一、zookeeper日志类型
zookeeper中有两类日志,分别是：

1、事务日志log
2、快照日志snapshot

事务日志 : 顾名思义，就是用于存放事务执行的相关信息，如zxid、cxid等。至于什么是zookeeper事务，放到后面的文章中讲。
快照日志 : ``zookeeper数据节点数据是运行在内存中的，当然内存保存这些结点的数据不可能无限大，而且数据节点的内容是动态变化的，因此zookeeper提供一中奖数据节点持久化的机制，每隔一段时间，zookeeper会将内存中的数据节点DataTree序列到磁盘中，因此就形成了我们的快照日志。
在zookeeper源码调试的过程中，诸如服务端的启动日志、客户端的启动日志、请求的相关日志，由于zookeeper采用log4j进行日志打印，因此log4j必须在类路径下查找log4j.properties文件，如果启动的过程中找不到该文件，则报错如下：
log4j:WARN No appenders could be found for logger (org.apache.log4j.jmx.HierarchyDynamicMBean).
log4j:WARN Please initialize the log4j system properly.
log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
复制代码因此，必须将log4j.properties放到类路径下，那么对于可恶的ant工程，类路径在哪里呢？我也不清楚，于是我直接在eclipse右键工程Build Path->Configure build path找到类路径为%baseDir%\src\java\main,具体如下图：

所以将conf目录下的log4j.properties拷贝一份至上图的目录中即可！
二、zookeeper日志可视化
上面说到zookeeper中有两种日志类型，但很遗憾，上面的日志你都无法直接看到，因为都是采用二进制进行保存的。查看zookeeper源代码也可以知道，zookeeper日志输出体现在FileTxnLog.java文件的append()方法中，具体如下：
// 写日志文件，采用log. + 哈希值的命名形式保存文件
logFileWrite = new File(logDir, ("log." + Long.toHexString(hdr.getZxid())));
// 文件输出流
fos = new FileOutputStream(logFileWrite);
// 采用BufferedOutputStream包裹fos成缓冲输出流
logStream=new BufferedOutputStream(fos);
// 这是重点，这里采用org.apache.jute.BinaryOutputArchive二进制构件对logStream进行包裹，输出二进制数据
oa = BinaryOutputArchive.getArchive(logStream);
复制代码对应的文件系统的文件如下图：

既然是二进制日志文件，那么我们直接打开该文件肯定是乱码嘛！怎么办呢？下面提供两种方法，这两种方法都是基于zookeeper提供的LogFormatter.java工具类来实现的。

1、在eclipse中开该类，然后运行该类的main方法的同时传入你想查看的日志文件路径即可
2、采用命令行java -classpath xxx.jar org.apache.zookeeper.server.LogFormatter logFilePath的形式进行查看

第一种方式：
这里假设我的日志文件存放路径为E:\\resources\\zookeeper-3.4.11\\conf\\log\\version-2\\log.1，当我在eclipse中运行main方法时，设置传入的参数即可，如下图：

然后就可以在控制台输出日志了，输出样例如下：
ZooKeeper Transactional Log File with dbid 0 txnlog format version 2
18-4-30 下午08时39分23秒 session 0x100022b44190000 cxid 0x0 zxid 0x1 createSession 30000

18-4-30 下午08时39分55秒 session 0x100022b44190000 cxid 0x0 zxid 0x2 closeSession null
EOF reached after 2 txns.
复制代码第二种方式：

1、在任意目录下新建一个临时目录,随便命名为logSee,将slf4j-api-1.6.1.jar和zookeeper-3.4.11.jar两个文件放到该目录下，然后打开命令行，执行java -classpath .;slf4j-api-1.6.1.jar;zookeeper-3.4.11.jar org.apache.zookeeper.server.LogFormatter E:\\resources\\zookeeper-3.4.11\\conf\\log\\version-2\\log.1

其中两个jar包之间采用分号;分隔而不是冒号，org.apache.zookeeper.server.LogFormatter表示LogFormatter.java的全路径命名(即包全名+类名),E:\\resources\\zookeeper-3.4.11\\conf\\log\\version-2\\log.1表示你想查看的日志文件。
输出如下：

三、zookeeper日志清理机制
zookeeper是将日志以文件的形式存放在磁盘中，久而久之，磁盘的文件就越来越多，zookeeper提供了一种定期清理日志和快照文件的机制。
QuorumPeerMain.java是zookeeper的启动类，在zookeeper启动之前会创建一个DatadirCleanupManager对象进行数据清理任务，DatadirCleanupManager会根据你配置文件的配置决定是否清理以及每隔多久进行清理，其底层原理采用JDK的Timer定时器实现，下面将对其实现机制从源码的角度进行分析：
先看QuorumPeerMain类，在其initializeAndRun()方法执行时会创建一个DatadirCleanupManager对象，并将zoo.cfg配置文件的相关配置传递给该对象，源码如下：
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
复制代码你可以在配置文件zoo.cfg文件中增加两个配置，分别是autopurge.snapRetainCount和autopurge.snapRetainCount 。autopurge.purgeInterval就是设置多少小时清理一次。而autopurge.snapRetainCount是设置保留多少个快照文件snapshot，之前的多有都删除。
有一点性能问题就是，一般zookeeper是运行在集群中，业务会比较繁忙，如果每隔多久去清理势必会影响性能，我们会想能否有一种在集群不繁忙的时候去执行清理操作，比如在每晚的12点。但是很遗憾，zookeeper并没有提供相应的实现，zookeeper采用Timer的方式去实现，而不是像quartz,Spring一样提供cron表达式配置。但注意，并不是说Timer不能实现指定时间执行，而是说zookeeper没有实现而已。因为quartz,Spring底层还是使用Timer和Executors去实现的嘛！
再来看看DatadirCleanupManager的start()方法：
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
复制代码现在原理一目了然了，日志源码分析就先到这了，同时你也阅读完了，非常棒！欢迎评论区留言，多多交流！

作者：拥抱心中的梦想
链接：https://juejin.cn/post/6844903600351608840
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
