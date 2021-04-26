---
layout:     post
title:      sentinel 是如何和 dashboard 通信的
subtitle:   对 sentinel 的学习笔记
date:       2021-04-20
author:     JerAxxxxxxx
header-img: img/background/sky_mks_trees_night_stars_art.jpg
catalog: true
tags:
    - sentinel
---


上一篇，学习了 sentiment-dashboard 都做了什么，其中很重要的一点就是：**向 sentinel 客户端同步流控规则**。那么，本篇就来学习一下，sentinel 客户端是如何与 dashboard 通信的。

#### Sentinel 客户端的初始化

这里我们先编写一个 demo 并引入 sentinel 的相关依赖。sentinel 主要使用方式就是通过 `SphU.entry("resourceName")` 来规定需要进行流控的资源，但其也封装了`@SentinelResource` 注解，可以通过切面的方式，方便在平时开发的使用。在 `SentinelResourceAspect` 类中，我们也可以看出，实质上还是调用了 `SphU.entry()` 方法。



![ SentinelResourceAspect类中的调用 ](/img/sentinel/sentinel_aspect.PNG)  



我们继续往下探索发现，`SphU.entry()` 调用的是 `Env.sph.entryWithType()` 方法。而这个 `Env` 类非常简单，包含了一个 `Sph` 对象和一个静态代码块。而真正的初始化方法也正在其中，便是 `InitExecutor.doInit()` 。

```java
public class Env {
    public static final Sph sph = new CtSph();
    static {
        // If init fails, the process will exit.
        InitExecutor.doInit();
    }
}
```

接下来就是 `doInit()` 代码，我去除了对于异常的处理：

```java
private static AtomicBoolean initialized = new AtomicBoolean(false);

public static void doInit() {
    	// 该 init 只会初始化一次
        if (!initialized.compareAndSet(false, true)) {
            return;
        }
        try {
            // 这里使用了 ServiceLoader 的方式，拿到 InitFunc 接口的所有实现类。
            ServiceLoader<InitFunc> loader = ServiceLoaderUtil.getServiceLoader(InitFunc.class);
            // OrderWrapper 有两个属性，分别是 InitFunc 实现类以及其优先级。
            List<OrderWrapper> initList = new ArrayList<OrderWrapper>();
            for (InitFunc initFunc : loader) {
                RecordLog.info("[InitExecutor] Found init func: " + initFunc.getClass().getCanonicalName());
                // 对 InitFunc 实现类的优先级进行排序
                insertSorted(initList, initFunc);
            }
            for (OrderWrapper w : initList) {
                // 调用每个实现类的 init() 方法
                w.func.init();
                RecordLog.info(String.format("[InitExecutor] Executing %s with order %d",
                    w.func.getClass().getCanonicalName(), w.order));
            }
        } catch (Exception ex) {
            // 异常处理及日志打印
        }
    }
```

可以看出，`doInit()` 方法主要是拿到 `InitFunc` 接口的所有实现类，根据 `@InitOrder` 注解对其优先级进行排序，如果不存在该注解，那么优先级将设置为最低级。最终调用所有实现类的 `init()` 方法。

![ InitFunc 的实现类 ](/img/sentinel/InitFunc_impls.png)  



这其中有两个类的优先级是最高的，分别是

- CommandCenterInitFunc

  主要负责 sentinel 客户端的启动及与 dashboard 的交互

- HeartbeatSenderInitFunc

  主要负责向 dashboard 发送心跳

本篇我们着重讲解与客户端的通信，即 `CommandCenterInitFunc` 做了什么。



#### CommandCenterInitFunc 做了什么

##### init

首先我们来看一下 `CommandCenterInitFunc` 的 `init()` 方法。

```java
public void init() throws Exception {
        CommandCenter commandCenter = CommandCenterProvider.getCommandCenter();
        if (commandCenter == null) {
            RecordLog.warn("[CommandCenterInitFunc] Cannot resolve CommandCenter");
            return;
        }
        commandCenter.beforeStart();
        commandCenter.start();
        RecordLog.info("[CommandCenterInit] Starting command center: "
                + commandCenter.getClass().getCanonicalName());
    }
```

获取 `CommandCenter` 实例，这里的便是 `SimpleHttpCommandCenter` 。`CommandCenterProvider` 中获取实例主要是通过 `SpiLoader.loadHighestPriorityInstance(CommandCenter.class);` 这一行代码来拿到优先级最高的实例。

```java
    public static <T> T loadHighestPriorityInstance(Class<T> clazz) {
        try {
            String key = clazz.getName();
            // Not thread-safe, as it's expected to be resolved in a thread-safe context.
            // SERVICE_LOADER_MAP 是一个 key 为类全限定名，value 为 ServiceLoader 对象。
            ServiceLoader<T> serviceLoader = SERVICE_LOADER_MAP.get(key);
            if (serviceLoader == null) {
                serviceLoader = ServiceLoaderUtil.getServiceLoader(clazz);
                SERVICE_LOADER_MAP.put(key, serviceLoader);
            }

            SpiOrderWrapper<T> w = null;
            for (T spi : serviceLoader) {
                // 解析优先级并封装进 SpiOrderWrapper 对象中
                int order = SpiOrderResolver.resolveOrder(spi);
                RecordLog.info("[SpiLoader] Found {} SPI: {} with order {}", clazz.getSimpleName(),
                    spi.getClass().getCanonicalName(), order);
                if (w == null || order < w.order) {
                    w = new SpiOrderWrapper<>(order, spi);
                }
            }
            return w == null ? null : w.spi;
        } catch (Throwable t) {
            RecordLog.error("[SpiLoader] ERROR: loadHighestPriorityInstance failed", t);
            t.printStackTrace();
            return null;
        }
    }
```

该方法在获取实例上，与上文中一样，均使用了 `ServiceLoader` 的方式，获取接口的实现类。也都对每个实现类进行了优先级的排序。

##### beforeStart

接下来我们来看 `beforeStart` 方法

```java
public void beforeStart() throws Exception {
    // Register handlers
    // 获取所有 handles
    Map<String, CommandHandler> handlers = CommandHandlerProvider.getInstance().namedHandlers();
    // 将 handlers 存入 SimpleHttpCommandCenter 类中的 handlerMap 中，其 map 的 kv 都相同
    registerCommands(handlers);
}
```

我们继续看 `CommandHandlerProvider` 类中的代码。

```java
private final ServiceLoader<CommandHandler> serviceLoader = ServiceLoaderUtil.getServiceLoader(
        CommandHandler.class);

public Map<String, CommandHandler> namedHandlers() {
        Map<String, CommandHandler> map = new HashMap<String, CommandHandler>();
    	// 遍历 ServiceLoader
        for (CommandHandler handler : serviceLoader) {
            String name = parseCommandName(handler);
            if (!StringUtil.isEmpty(name)) {
                map.put(name, handler);
            }
        }
        return map;
    }


private String parseCommandName(CommandHandler handler) {
        CommandMapping commandMapping = handler.getClass().getAnnotation(CommandMapping.class);
        if (commandMapping != null) {
            return commandMapping.name();
        } else {
            return null;
        }
    }
```

不难看出 `CommandHandlerProvider`  主要获取所有所有 handles，并将他们封装进一个 key 是命令名，value 为实例的 map 中。

而这里 handles 具体是什么呢，我们通过查看 `@CommandMapping` 的类发现，handles 即是 sentinel 客户端对 dashboard 提供的接口。





##### start

接下来就是最后的 `start()` 方法了

```java
public void start() throws Exception {
    	// 获取 cpu 核心数的线程数
        int nThreads = Runtime.getRuntime().availableProcessors();
        this.bizExecutor = new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS,
            new ArrayBlockingQueue<Runnable>(10),
            new NamedThreadFactory("sentinel-command-center-service-executor"),
            new RejectedExecutionHandler() {
                @Override
                public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
                    CommandCenterLog.info("EventTask rejected");
                    throw new RejectedExecutionException();
                }
            });

        Runnable serverInitTask = new Runnable() {
            int port;

            {
                try {
                    port = Integer.parseInt(TransportConfig.getPort());
                } catch (Exception e) {
                    // 若没有指定端口，就采用默认的 8719 端口
                    port = DEFAULT_PORT;
                }
            }

            @Override
            public void run() {
                boolean success = false;
                ServerSocket serverSocket = getServerSocketFromBasePort(port);

                if (serverSocket != null) {
                    CommandCenterLog.info("[CommandCenter] Begin listening at port " + serverSocket.getLocalPort());
                    socketReference = serverSocket;
                    // 异步执行 ServerThread() 的任务
                    executor.submit(new ServerThread(serverSocket));
                    success = true;
                    port = serverSocket.getLocalPort();
                } else {
                    CommandCenterLog.info("[CommandCenter] chooses port fail, http command center will not work");
                }

                if (!success) {
                    port = PORT_UNINITIALIZED;
                }

                TransportConfig.setRuntimePort(port);
                executor.shutdown();
            }

        };

        new Thread(serverInitTask).start();
    }
```

这个方法有些长，我们拆解出来看。首先创建一个线程数为 cpu 核心数的线程池。获取端口后，创建一个 ServerSocket 对象，我们看一下 `getServerSocketFromBasePort` 方法，该方法会尝试创建 ServerSocket  对象三次，如果不成功，将会把传入的端口 +1 ，并继续尝试创建。

```java
private static ServerSocket getServerSocketFromBasePort(int basePort) {
        int tryCount = 0;
        while (true) {
            try {
                ServerSocket server = new ServerSocket(basePort + tryCount / 3, 100);
                server.setReuseAddress(true);
                return server;
            } catch (IOException e) {
                tryCount++;
                try {
                    TimeUnit.MILLISECONDS.sleep(30);
                } catch (InterruptedException e1) {
                    break;
                }
            }
        }
        return null;
    }
```

其次，异步执行 `executor.submit(new ServerThread(serverSocket));` 任务，该线程主要设置了 socket 的超时时间，并使用 `SimpleHttpCommandCenter` 类中的 `ExecutorService` 对象，bizExecutor 来执行 `HttpEventTask` 的任务。 `HttpEventTask`  的 run 方法比较长，我截取一部分：

```java
            long start = System.currentTimeMillis();
			// .........
			// 构造请求的 url
			CommandRequest request = processQueryString(firstLine);
			// 获取在 beforeStart 设置的 handles
            CommandHandler<?> commandHandler = SimpleHttpCommandCenter.getHandler(commandName);
            if (commandHandler != null) {
                // 调用 handle 方法执行
                CommandResponse<?> response = commandHandler.handle(request);
                handleResponse(response, printWriter);
            } else {
                // No matching command handler.
                writeResponse(printWriter, StatusCode.BAD_REQUEST, "Unknown command `" + commandName + '`');
            }

            long cost = System.currentTimeMillis() - start;
```

该线程主要去调度了每一个 `CommandHandler` 实例，这样也保证了 sentinel 客户端的启动。

#### 总结

至此，sentinel 客户端的启动大致也分析完毕。客户端启动后，通过 restful 对外暴露接口，建立与 dashboard 之间的通信，保证了在默认情况下 dashboard 中的改动可以在客户端得到同步。当然，在生产环境中，我们不会让规则都写入内存中，这样，服务重启后会丢掉所有的规则，通常会通过 配置中心/注册中心 进行保存。

----

<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-sa/4.0/88x31.png" /></a><br />本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">知识共享署名-相同方式共享 4.0 国际许可协议</a>进行许可。
