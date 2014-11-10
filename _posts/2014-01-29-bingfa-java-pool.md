---
layout: post
title: java线程池的实现原理
category: 并发编程
---

####	1. java 线程池创建
  ThreadPoolExecutor来创建一个线程池，java并发包内几个线程池都是通过调整如下方法的参数而来。

>ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,TimeUnit unit,BlockingQueue<Runnable> workQueue,ThreadFactory threadFactory,RejectedExecutionHandler handler)

####	2. 参数解读
  
1.	corePoolSize（线程池的基本大小）：当提交一个任务到线程池时，线程池会创建一个线程来执行任务，即使其他空闲的基本线程能够执行新任务也会创建线程，
	等到需要执行的任务数大于线程池基本大小时就不再创建。如果调用了线程池的prestartAllCoreThreads方法，线程池会提前创建并启动所有基本线程。
2.	maximumPoolSize（线程池最大大小）：线程池允许创建的最大线程数。如果队列满了，并且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务.
	值得注意的是如果使用了无界队列这个参数就没什么效果。
3.	keepAliveTime（线程活动保持时间）：线程池的工作线程空闲后，保持存活的时间。所以如果任务很多，并且每个任务执行的时间比较短，可以调大这个时间，提高线程的利用率。
4.	TimeUnit（线程活动保持时间的单位）：可选的单位有天（DAYS），小时（HOURS），分钟（MINUTES），毫秒(MILLISECONDS)，微秒(MICROSECONDS, 千分之一毫秒)和
	毫微秒(NANOSECONDS, 千分之一微秒)。
5.	workQueue（任务队列）：用于保存等待执行的任务的阻塞队列。 可以选择以下几个阻塞队列。
	* ArrayBlockingQueue：是一个基于数组结构的有界阻塞队列，此队列按 FIFO（先进先出）原则对元素进行排序。
	* LinkedBlockingQueue：一个基于链表结构的阻塞队列，此队列按FIFO （先进先出） 排序元素。静态工厂方法Executors.newFixedThreadPool()使用了这个队列，队列长度为int最大值，可以认为是无界队列。
	* SynchronousQueue：缓冲区为一的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，静态工厂方法Executors.newCachedThreadPool使用了这个队列。
	* PriorityBlockingQueue：一个具有优先级的无限阻塞队列。
6.	ThreadFactory：用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程设置更有意义的名字。
7.	RejectedExecutionHandler（饱和策略）：当队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。这个策略默认情况下是AbortPolicy，表示无法处理新任务时抛出异常。以下是JDK1.5提供的四种策略。
	* AbortPolicy：直接抛出异常。
	* CallerRunsPolicy：只用调用者所在线程来运行任务。
	* DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务。
	* DiscardPolicy：不处理，丢弃掉。
	* 当然也可以根据应用场景需要来实现RejectedExecutionHandler接口自定义策略。如记录日志或持久化不能处理的任务。

####	3.执行过程 
	1. 首先线程池判断基本线程池是否已满？没满，创建一个工作线程来执行任务。满了，则进入下个流程。
	2. 其次线程池判断工作队列是否已满？没满，则将新提交的任务存储在工作队列里。满了，则进入下个流程。
	3. 最后线程池判断整个线程池是否已满？没满，则创建一个新的工作线程来执行任务，满了，则交给饱和策略来处理这个任务。
	源码分析：
	public void execute(Runnable command) {
	    if (command == null)
	       throw new NullPointerException();
	    //如果线程数小于基本线程数，则创建线程并执行当前任务 
	    if (poolSize >= corePoolSize || !addIfUnderCorePoolSize(command)) {
	    //如线程数大于等于基本线程数或线程创建失败，则将当前任务放到工作队列中。
	        if (runState == RUNNING && workQueue.offer(command)) {
	            if (runState != RUNNING || poolSize == 0)
	                      ensureQueuedTaskHandled(command);
	        }
	    //如果线程池不处于运行中或任务无法放入队列，并且当前线程数量小于最大允许的线程数量，
	则创建一个线程执行任务。
	        else if (!addIfUnderMaximumPoolSize(command))
	        //抛出RejectedExecutionException异常
	            reject(command); // is shutdown or saturated
	    }
	}

####	4.可能会犯的错
  初始corePoolSize,maximumPoolSize 不同，任务队列却用了无界队列，这样会导致线程数达到corePoolSize值时所有的任务都会放进任务队列。
  请求数达到边界时既不能充分调用CPU，又可能因为无界队列任务过多导致内存溢出，特别是遇到类似DDOS攻击的情况。通常WEB服务器的设计为了
  充分实现并发并且不丢失请求，会固定线程池的大小，启用无界队列，但这样也给请求峰值留下了内存溢出的隐患，所以如何设置还需要慎重考虑。
 如果类似广告系统允许系统丢失一些请求，但需要无宕机正常服务99%的用户，设置成有界队列也是一种不错的选择，如何设置还要看服务的性质。

>如restlet在整合jetty的时候默认采用的无界队列，而且封装过程不对外开放。
在遇到的DDOS攻击时会由于内存溢出导致宕机，在出现特殊情况如果允许丢部分请求，可以设置有限队列，只能通过反射启动时动态修改配置信息
{% highlight java %}
public static void main(String[] args) throws Exception {
          System.setProperty("org.restlet.engine.loggerFacadeClass", "org.restlet.ext.slf4j.Slf4jLoggerFacade");
          CommonUtils.loadLogbackConfiguration(CommonUtils.__CONF_DIR__);
          LOGGER.info("TotalMemory:" + Runtime.getRuntime().totalMemory() / (1024 * 1024) + " M");

          final org.restlet.Component component = new org.restlet.Component();
          Injector injector = RestletGuice.createInjector(new GuiceModule());
          Configuration configuration = injector.getInstance(Configuration.class);

          Application application = new Application(injector);
          component.getDefaultHost().attach("/feedback", application);
          Server server = component.getServers().add(Protocol.HTTP, configuration.getInt("server.port"));

          // jetty config
          server.getContext().getParameters().add("minThreads", "50");
          server.getContext().getParameters().add("maxThreads", "1536");
          server.getContext().getParameters().add("acceptorThreads", "4");
          server.getContext().getParameters().add("gracefulShutdown", "5000");
          server.getContext().getParameters().add("useForwardedForHeader", "true");
          component.start();
          //下面代码解决restlet 2.0 版本线程队列过长（int 最大值），对jetty整合有问题，很多参数没办法外边配置了，导致请求量过载或者遇到DDOS攻击 直接挂掉，不会扔请求
          Class clazz = server.getClass();
          Method m = clazz.getDeclaredMethod("getHelper");
          m.setAccessible(true);
          org.restlet.ext.jetty.HttpServerHelper helper = (org.restlet.ext.jetty.HttpServerHelper) m.invoke(server);

          m = JettyServerHelper.class.getDeclaredMethod("getWrappedServer");
          m.setAccessible(true);
          org.eclipse.jetty.server.Server s = (org.eclipse.jetty.server.Server) m.invoke(helper);

          m = s.getClass().getMethod("getThreadPool");
          m.setAccessible(true);
          QueuedThreadPool pool = (QueuedThreadPool) m.invoke(s);
          Field f = pool.getClass().getDeclaredField("_jobs");
          f.setAccessible(true);
          f.set(pool, new ArrayBlockingQueue<Runnable>(50000));

          LOGGER.info("Sever started on " + Inet4Address.getLocalHost().getHostAddress() + ":" + configuration.getInt("server.port"));
          Runtime.getRuntime().addShutdownHook(new Thread() {
               public void run() {
                    try {
                         component.stop();
                         LOGGER.info("gracefully shutdown feedback system");
                    } catch (Exception e) {
                         e.printStackTrace();
                    }
               }
          });
          SignalManager.install();
     }
{% endhighlight %}
 
[参考：JAVA线程池的分析和使用](http://www.infoq.com/cn/articles/java-threadPool?utm_source=infoq&utm_medium=related_content_link&utm_campaign=relatedContent_articles_clk)



 