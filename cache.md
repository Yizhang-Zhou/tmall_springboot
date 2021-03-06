## 接口定义
1. AutoReloadCache<K, V>作为一切缓存的抽象类，成员：
   - `Map<K,V>cache`,（需要考虑。如果数据库没有数据，该怎么返回）
   - `initialed`，是否初始化。如果没有，作为首次访问数据，加载数据
   - `Map<K,V>reload()`方法,直接从数据库加载数据。作为具体加载方式，由子类完成重写任务。
   - `Map<K,V>get()`方法,返回当前缓存数据，类似，`V get(K key)`,根据key来获取缓存内容
   - `getDelay()`方法，默认间隔多少次从数据库取数据
   - `checkAndStartTimer()`方法，如果没有初始化，启动线程。利用newSingleThreadScheduledExecutor线程池的scheduleWithFixedDelay方法来完成reload的任务。
      - scheduleWithFixedDelay的几个参数：
        - Runnable command,要执行的任务
        - long initialDelay,首次执行的延迟时间
        - delay ,一次执行终止和下一次执行开始之间的延迟,getDelay()
        - unit,initialDelay 和 delay 参数的时间单位,TimeUnit.SECONDS
      - scheduleWithFixedDelay该方法描述：创建并执行一个在给定初始延迟后首次启用的定期操作，随后，在每一次执行终止和下一次执行开始之间都存在给定的延迟。如果任务的任一执行遇到异常，就会取消后续执行。否则，只能通过执行程序的取消或终止方法来终止该任务。
      - newSingleThreadScheduledExecutor可以在构造函数中传入ThreadFactory，自己可以定义一个CacheThreadFactory,在其构造函数中传入函数的名字，以及定义异常处理等（先加入线程名）
      
2. 定义CacheService服务接口,根据具体业务，加入具体业务的接口方法。在其实现类中，注入具体DAO,主要在于：成员变量为具体实现缓存的匿名类对象，也就是AutoReloadCache<K, V>的对象，并在里面自行完成reload()方法的重写，根据不同业务，重写加载函数。
3. 使用方式：在具体业务的service类中，调用CacheService中定义的各种具体业务方法（２中已重写reload方法，在该方法中完成对数据的读）。
4. 目前只能重加载不需要前台参数的数据库的内容。后面内容，学会添加参数后，如何使用缓存。以及map到底怎么在存。
   - 当前reload是一次性将所有数据查完，然后统一一次性放入map.
   - 目前出现的参数，也是根据其他cache的获取查询参数，如时间的获取。时间的缓存。还没遇到前台传来的值用于查询。
   - 前台确实可以传，但后台已经把缓存做好了。适用于时间变化。


## 实现中遇到的问题：
1. 首先是缓存代码中缓存的值一直没有更新，就怀疑线程是不是挂了，本想使用jconsole控制台来查看线程运行情况，由于项目部署在远程服务器，始终连不上（中间好像是有一层代理，还是端口没有开放的原因），于是不清楚线程是否挂的情况下，查看日志，并没有发现报错。在Java中，线程中的异常是不能抛出到调用该线程的外部方法中捕获的。而这里缓存的线程通过线程工厂创建的，里面重写了UncaughtExceptionHanlder方法。理论上，如果报错应该是能捕获到的。
1. 一般线程的异常，可以通过重写setUncaughtExceptionHandler方法来捕获。
2. [线程池](https://www.jianshu.com/p/281958d20b04)（特指ThreadPoolExecutor）中线程执行时，有2种方法execute(Runnable) 和submit(Callable/Runnable)。使用 execute 方法执行任务所抛出的异常可以捕获UncaughtExceptionHandler。
   1. submit提交任务线程时，在任务线程中重写setUncaughtExceptionHandler方法后，该方法不会捕获异常。原因如下源码，可以知道，如果想知道 submit 的执行结果是成功还是失败，必须调用 Future.get() 方法。
   2. submit的实现在[AbstractExecutorService中](https://github.com/xuzhuang1996/MyJava/blob/master/%E5%A4%9A%E7%BA%BF%E7%A8%8B/%E5%A4%9A%E7%BA%BF%E7%A8%8B.md#threadpoolexecutor)，而ThreadPoolExecutor继承了这种默认实现。
   
            //submit 方法是调用 execute 实现任务执行的。但是在调用 execute 之前，任务会被封装进 FutureTask 类中，
            //然后最终工作线程执行的是 FutureTask 中的 run 方法。
            public Future<?> submit(Runnable task) {
                if (task == null) throw new NullPointerException();
                RunnableFuture<Void> ftask = newTaskFor(task, null);
                execute(ftask);
                return ftask;
            }

            //而调用 submit 方法后也就是进入FutureTask.run。如果任务抛出异常，会被 setException 方法赋给代表执行结果的 outcome 变量，
            //而不会继续抛出。因此，UncaughtExceptionHandler 也没有机会处理。
            //FutureTask.run
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }

            protected void setException(Throwable t) {
                if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
                    outcome = t;
                    UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
                    finishCompletion();
                }
            }
3. ScheduledThreadPoolExecutor是我在做缓存时使用到的线程池。对于 ScheduledThreadPoolExecutor.scheduleWithFixedDelay 和 scheduleAtFixedRate 这两个方法（循环执行，获取结果只能停止循环），其返回的 Future 只会用来取消任务，而不是得到结果。对于这两个方法来说，在 Runnable.run 方法中加 try...catch 是必须的，否则很有可能出错了却毫不知情。 
4. 用hashmap实现缓存的缺点：
   1. redis 数据可持久化保存，有些缓存你想重启程序后还能继续使用，map实现不了
   2. redis 可以实现分布式部署，map实现不了
   3. hashmap不是线程安全的，可以concurrentHashmap
   4. Redis 缓存有过期机制，Map 本身无此功能
