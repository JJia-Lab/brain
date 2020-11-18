# 揭秘nacos-config客户端工作原理

​	随着SpringCloud中的许多组件逐渐的不再维护，越来越多的人开始使用SpringCloud Alibaba框架，SpringCloud Alibaba是由阿里巴巴开发的开源微服务全家桶，事实上，SpringCloud Alibaba很可能成为SpringCloud第二代的标准实现，而SpringCloud Alibaba有着很多的组件，比如：Sentinel，Nacos，Rockermq等等，其中，Nacos组件可以替代SpringCloud中的Eureka作为注册中心，同时也可以替代SpringCloud中的SpringCloudConfig作为分布式配置中心，本文就来初探nacos作为分布式配置中心时的工作原理。

## 一、什么是分布式配置中心

​	随着业务的发展、微服务架构的升级，服务的数量、程序的配置日益增多（各种微服务、各种服务器地址、各种参数），传统的配置文件方式和数据库的方式已无法满足开发人员对配置管理的要求，因此，我们需要配置中心来统一管理配置！把业务开发者从复杂以及繁琐的配置中解脱出来，只需专注于业务代码本身，从而能够显著提升开发以及运维效率。

## 二、什么是nacos

[Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。Nacos 帮助您更敏捷和容易地构建、交付和管理微服务平台。 Nacos 是构建以“服务”为中心的现代应用架构 (例如微服务范式、云原生范式) 的服务基础设施。]: https://nacos.io/zh-cn/docs/what-is-nacos.html

## 三、配置环境、运行demo

​	根据nacos官方文档中的，先下载nacos，配置nacos数据库文件，随后使用启动命令启动nacos-server

```bat
cmd startup.cmd -m standalone
```

​	打开localhost:8848/nacos/index.html，看到以下界面，代表启动成功

[![DeLFYt.png](https://s3.ax1x.com/2020/11/18/DeLFYt.png)](https://imgchr.com/i/DeLFYt)

​	接下来，根据官方文档中的示例代码，建立controller

```java
@RestController
@RefreshScope(proxyMode = DEFAULT)//动态刷新
@Slf4j
public class ConfigController {
    /**
     * 从配置中心获取字段的值，具有动态刷新的效果
     */
    @Value("${config.info}")
    private String configInfo;
    @Value("${number}")
    private String number;
    @RequestMapping("/config")
    private String getConfigInfo(){
        log.info("configInfo:{}",configInfo);
        log.info("number:{}",number);
        return configInfo+"======"+number;
    }
}
```

​	建立bootstrap.yml和application.yml（bootstrap的执行优先级比application高），之所以需要配置 `spring.application.name` ，是因为它是构成 Nacos 配置管理 `dataId`字段的一部分，在Nacos Spring Cloud 中，`dataId` 的完整格式如下：

```
${prefix}-${spring.profiles.active}.${file-extension}
```

bootstrap.yml

```yaml
server:
  port: 3377

spring:
  application:
    name: nacos-config
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
      # 配置中心
      config:
        server-addr: localhost:8848
        # 指定配置文件yaml
        file-extension: yaml
        # 指定组名，默认是DEFAULT_GROUP
        group: DEFAULT_GROUP
        # 命名空间，默认是public
        namespace: public

# 服务名-active环境.配置文件类型 如：nacos-config-dev.yaml
```

application.yml

```yaml
spring:
  profiles:
    active: dev
```

​	配置完yml后，启动示例项目，同时，在nacos控制台中创建一个配置，DataId为nacos-config-dev.yaml，配置内容

```yaml
config:
    info: yes,it is config
number: 100
```

​	配置完成后点击发布，随后访问http://localhost:3377/config，可以看到输出了

[![DeO2vT.png](https://s3.ax1x.com/2020/11/18/DeO2vT.png)](https://imgchr.com/i/DeO2vT)

​	同时控制台上输出了：

```
2020-11-18 11:41:56.888  INFO 6376 --- [nio-3377-exec-5] jiajia.iot.controller.ConfigController   : configInfo:yes,it is config
2020-11-18 11:41:56.888  INFO 6376 --- [nio-3377-exec-5] jiajia.iot.controller.ConfigController   : number:100
```

​	随后，修改控制台中配置内容number为200，点击发布，控制台输出

```
2020-11-18 11:44:28.010  INFO 6376 --- [ost_8848-public] o.s.c.e.event.RefreshEventListener       : Refresh keys changed: [number]
```

​	代表配置已刷新。

​	通过示例项目我们可以发现，nacos控制台上发布的配置，client端会立即收到信息，并且修改了client端的配置，接下来我们就来开始分析其中的原理

## 四、原理分析

​	要想分析nacos-config源码，就先要找到nacos的主入口，从官方文档的JAVA SDK部分可以知道，服务启动时从nacos获取配置的方法是ConfigService的getConfig()方法，由于示例工程使用的是spring-cloud-starter-alibaba-nacos-config，所以我们不需要用这种方式来启动服务，但是我们知道了入口，于是我定位到ConfigService，发现它是一个接口，且这个接口只有一个实现类NacosConfigService，位于com.alibaba.nacos.client.config包下，由于SpringBoot的AutoConfig机制，我们知道启动后肯定会执行NacosConfigService的初始化方法，于是我就将断点打在NacosConfigService的构造函数上

```java
public NacosConfigService(Properties properties) throws NacosException {
        String encodeTmp = properties.getProperty(PropertyKeyConst.ENCODE);
        if (StringUtils.isBlank(encodeTmp)) {
            encode = Constants.ENCODE;
        } else {
            encode = encodeTmp.trim();
        }
        initNamespace(properties);
    	//创建Http代理
        agent = new MetricsHttpAgent(new ServerHttpAgent(properties));
        agent.start();
    	//创建ClientWorker，客户端工作的核心方法，接下来要重点分析的地方
        worker = new ClientWorker(agent, configFilterChainManager, properties);
    }
```

​	我们可以看到在NacosConfigService的构造函数中初始化了一个HttpAgent和一个ClientWorker，那他们分别是用来做什么的呢，先来看HttpAgent

[![DmCwrt.png](https://s3.ax1x.com/2020/11/18/DmCwrt.png)](https://imgchr.com/i/DmCwrt)

​	通过类的结构我们可以看到这个类就是代理了HttpAgent类的一些方法，翻看代码发现其实agent是装饰器模式实现的，实际执行的类是ServerHttpAgent类，在com.alibaba.nacos.client.config.http包下，agent实际上是在ClientWorker中发挥作用的，接下来我们来看ClientWorker，首先看构造函数

​	

```java
public ClientWorker(final HttpAgent agent, final ConfigFilterChainManager configFilterChainManager, final Properties properties) {
    	//持有agent对象
        this.agent = agent;
        this.configFilterChainManager = configFilterChainManager;

        // Initialize the timeout parameter

        init(properties);

        executor = Executors.newScheduledThreadPool(1, new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r);
                t.setName("com.alibaba.nacos.client.Worker." + agent.getName());
                t.setDaemon(true);
                return t;
            }
        });
		//长轮询的拉取线程
        executorService = Executors.newScheduledThreadPool(Runtime.getRuntime().availableProcessors(), new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r);
                t.setName("com.alibaba.nacos.client.Worker.longPolling." + agent.getName());
                t.setDaemon(true);
                return t;
            }
        });
		//定时任务，每隔10ms调用一次checkConfigInfo()方法
        executor.scheduleWithFixedDelay(new Runnable() {
            @Override
            public void run() {
                try {
                    checkConfigInfo();
                } catch (Throwable e) {
                    LOGGER.error("[" + agent.getName() + "] [sub-check] rotate check error", e);
                }
            }
        }, 1L, 10L, TimeUnit.MILLISECONDS);
    }
```

​	可以看到ClientWorker初始化时启动了两个线程池，一个做长轮询拉取线程，另一个是定时任务调度线程，周期性的调用checkConfigInfo()方法，这个方法是客户端处理核心

​	接下来我们来看checkConfigInfo()到底做了什么

```java
 public void checkConfigInfo() {
        // 分任务
        int listenerSize = cacheMap.get().size();
        // 向上取整为批数
        int longingTaskCount = (int) Math.ceil(listenerSize / ParamUtil.getPerTaskConfigSize());
        if (longingTaskCount > currentLongingTaskCount) {
            for (int i = (int) currentLongingTaskCount; i < longingTaskCount; i++) {
                // 要判断任务是否在执行 这块需要好好想想。 任务列表现在是无序的。变化过程可能有问题
                executorService.execute(new LongPollingRunnable(i));
            }
            currentLongingTaskCount = longingTaskCount;
        }
    }
```

​	可以看到checkConfigInfo方法是执行一批任务，然后给executorService去执行，执行的任务就是LongPollingRunnable，那么LongPollingRunnable又是做什么的呢，继续往下看

```java
@Override
        public void run() {

            List<CacheData> cacheDatas = new ArrayList<CacheData>();
            List<String> inInitializingCacheList = new ArrayList<String>();
            //...
			//省略部分代码
                // check server config
            	// 通过长轮询的方式，从服务端获取data变化的dataId
                // 默认是30s超时
                List<String> changedGroupKeys = checkUpdateDataIds(cacheDatas, inInitializingCacheList);
				// 获得变化的dataId清单之后，循环从server端查询最新的配置信息，并更新到本地的缓存文件中
                for (String groupKey : changedGroupKeys) {
                    String[] key = GroupKey.parseKey(groupKey);
                    String dataId = key[0];
                    String group = key[1];
                    String tenant = null;
                    if (key.length == 3) {
                        tenant = key[2];
                    }
                    try {
                        //此处开始从server端查询配置信息，并更新到CacheData中
                        String content = getServerConfig(dataId, group, tenant, 3000L);
                        CacheData cache = cacheMap.get().get(GroupKey.getKeyTenant(dataId, group, tenant));
                        //将新配置信息放入CacheData中并重新生成md5
                        cache.setContent(content);
                        LOGGER.info("[{}] [data-received] dataId={}, group={}, tenant={}, md5={}, content={}",
                            agent.getName(), dataId, group, tenant, cache.getMd5(),
                            ContentUtils.truncateContent(content));
                    } catch (NacosException ioe) {
                        String message = String.format(
                            "[%s] [get-update] get changed config exception. dataId=%s, group=%s, tenant=%s",
                            agent.getName(), dataId, group, tenant);
                        LOGGER.error(message, ioe);
                    }
                }
                for (CacheData cacheData : cacheDatas) {
                    if (!cacheData.isInitializing() || inInitializingCacheList
                        .contains(GroupKey.getKeyTenant(cacheData.dataId, cacheData.group, cacheData.tenant))) {
                        cacheData.checkListenerMd5();
                        cacheData.setInitializing(false);
                    }
                }
                inInitializingCacheList.clear();

                executorService.execute(this);

            } catch (Throwable e) {

                // If the rotation training task is abnormal, the next execution time of the task will be punished
                LOGGER.error("longPolling error : ", e);
                executorService.schedule(this, taskPenaltyTime, TimeUnit.MILLISECONDS);
            }
        }
```

```java
/**
     * 从Server获取值变化了的DataID列表。返回的对象里只有dataId和group是有效的。 保证不返回NULL。
     */
	// 拼接约定格式的参数字符串，服务端会按照约定的格式进行解析
    // 会将group，dataId，content拼接到参数中
    List<String> checkUpdateDataIds(List<CacheData> cacheDatas, List<String> inInitializingCacheList) throws IOException {
        StringBuilder sb = new StringBuilder();
        for (CacheData cacheData : cacheDatas) {
            if (!cacheData.isUseLocalConfigInfo()) {
                sb.append(cacheData.dataId).append(WORD_SEPARATOR);
                sb.append(cacheData.group).append(WORD_SEPARATOR);
                if (StringUtils.isBlank(cacheData.tenant)) {
                    sb.append(cacheData.getMd5()).append(LINE_SEPARATOR);
                } else {
                    sb.append(cacheData.getMd5()).append(WORD_SEPARATOR);
                    sb.append(cacheData.getTenant()).append(LINE_SEPARATOR);
                }
                if (cacheData.isInitializing()) {
                    // cacheData 首次出现在cacheMap中&首次check更新
                    inInitializingCacheList
                        .add(GroupKey.getKeyTenant(cacheData.dataId, cacheData.group, cacheData.tenant));
                }
            }
        }
        boolean isInitializingCacheList = !inInitializingCacheList.isEmpty();
        return checkUpdateConfigStr(sb.toString(), isInitializingCacheList);
    }
```

```java
/**
     * 从Server获取值变化了的DataID列表。返回的对象里只有dataId和group是有效的。 保证不返回NULL。
     */
    List<String> checkUpdateConfigStr(String probeUpdateString, boolean isInitializingCacheList) throws IOException {

        List<String> params = Arrays.asList(Constants.PROBE_MODIFY_REQUEST, probeUpdateString);

        List<String> headers = new ArrayList<String>(2);
        headers.add("Long-Pulling-Timeout");
        headers.add("" + timeout);

        // told server do not hang me up if new initializing cacheData added in
        if (isInitializingCacheList) {
            headers.add("Long-Pulling-Timeout-No-Hangup");
            headers.add("true");
        }

        if (StringUtils.isBlank(probeUpdateString)) {
            return Collections.emptyList();
        }

        try {
            //这里就是所谓的长轮询
            // url:http://ip:port/v1/cs/configs/listener
        	// 超时时间是默认值30s
            HttpResult result = agent.httpPost(Constants.CONFIG_CONTROLLER_PATH + "/listener", headers, params,
                agent.getEncode(), timeout);

            if (HttpURLConnection.HTTP_OK == result.code) {
                setHealthServer(true);
                //将解析好的dataId返回到上一层
                return parseUpdateDataIdResponse(result.content);
            } else {
                setHealthServer(false);
                LOGGER.error("[{}] [check-update] get changed dataId error, code: {}", agent.getName(), result.code);
            }
        } catch (IOException e) {
            setHealthServer(false);
            LOGGER.error("[" + agent.getName() + "] [check-update] get changed dataId exception", e);
            throw e;
        }
        return Collections.emptyList();
    }
```

​	我们看到，LongPollingRunnable就是执行长轮询，检查服务端哪些dataId发生了变化，然后通过得到的dataId到Server端查询配置信息，更新到CacheData中，可以看到在上面的代码中，CacheData一直出现，CacheData在这里是起什么作用的呢，一起来看一下

[![DmAE9S.png](https://s3.ax1x.com/2020/11/18/DmAE9S.png)](https://imgchr.com/i/DmAE9S)

​	通过这些成员变量我们可以看到除了name，dataId，group这三个配置相关的属性，还有taskId，md5，listener等属性。其中Listener对象由ManagerListenerWrap类所包装，而md5则是通过当前对象的content计算得出的md5值。

​	我们回过头继续往下，可以看到执行了cacheData.checkListenerMd5()方法，我们来看它做了什么

```java
void checkListenerMd5() {
    for (ManagerListenerWrap wrap : listeners) {
        //如果CacheData当前的md5与CacheData持有的所有Listener中保存的md5的值不一致，则执行一个安全的通知listener的方法
        if (!md5.equals(wrap.lastCallMd5)) {
            safeNotifyListener(dataId, group, content, md5, wrap);
        }
    }
}

private void safeNotifyListener(final String dataId, final String group, final String content,
                                final String md5, final ManagerListenerWrap listenerWrap) {
    final Listener listener = listenerWrap.listener;

    Runnable job = new Runnable() {
        @Override
        public void run() {
            ClassLoader myClassLoader = Thread.currentThread().getContextClassLoader();
            ClassLoader appClassLoader = listener.getClass().getClassLoader();
            try {
                if (listener instanceof AbstractSharedListener) {
                    AbstractSharedListener adapter = (AbstractSharedListener) listener;
                    adapter.fillContext(dataId, group);
                    LOGGER.info("[{}] [notify-context] dataId={}, group={}, md5={}", name, dataId, group, md5);
                }
                // 执行回调之前先将线程classloader设置为具体webapp的classloader，以免回调方法中调用spi接口是出现异常或错用（多应用部署才会有该问题）。
                Thread.currentThread().setContextClassLoader(appClassLoader);

                ConfigResponse cr = new ConfigResponse();
                cr.setDataId(dataId);
                cr.setGroup(group);
                cr.setContent(content);
                configFilterChainManager.doFilter(null, cr);
                String contentTmp = cr.getContent();
                listener.receiveConfigInfo(contentTmp);
                listenerWrap.lastCallMd5 = md5;
                LOGGER.info("[{}] [notify-ok] dataId={}, group={}, md5={}, listener={} ", name, dataId, group, md5,
                    listener);
            } catch (NacosException de) {
                LOGGER.error("[{}] [notify-error] dataId={}, group={}, md5={}, listener={} errCode={} errMsg={}", name,
                    dataId, group, md5, listener, de.getErrCode(), de.getErrMsg());
            } catch (Throwable t) {
                LOGGER.error("[{}] [notify-error] dataId={}, group={}, md5={}, listener={} tx={}", name, dataId, group,
                    md5, listener, t.getCause());
            } finally {
                Thread.currentThread().setContextClassLoader(myClassLoader);
            }
        }
    };

    final long startNotify = System.currentTimeMillis();
    try {
        if (null != listener.getExecutor()) {
            listener.getExecutor().execute(job);
        } else {
            job.run();
        }
    } catch (Throwable t) {
        LOGGER.error("[{}] [notify-error] dataId={}, group={}, md5={}, listener={} throwable={}", name, dataId, group,
            md5, listener, t.getCause());
    }
    final long finishNotify = System.currentTimeMillis();
    LOGGER.info("[{}] [notify-listener] time cost={}ms in ClientWorker, dataId={}, group={}, md5={}, listener={} ",
        name, (finishNotify - startNotify), dataId, group, md5, listener);
}
```

​	我们看到调用了safeNotifyListener()方法，在safeNotifyListener()方法中，最重要的是listener.receiveConfigInfo()方法，这个方法就是监听Server端配置的改变并触发receiveConfigInfo()方法，这个方法的实现我们先不看，现在我们需要回过头看看当ClientWorker的初始化结束之后会做些什么，通过断点跟踪我们发现

走到了com.alibaba.nacos.api.config.ConfigService#addListener方法里，最终往CacheData中插入了一个Listener

[![DmYt76.png](https://s3.ax1x.com/2020/11/18/DmYt76.png)](https://imgchr.com/i/DmYt76)

​	我们看到监听的是nacos-config-dev.yaml这个dataId，DEFAULT_GROUP这个group。到这里，整个启动流程的主要部分就已经结束了，接下来回过头来看看在config发生变化时的流程，在上面我们已经看到了LongPollingRunnable这个长轮询定时任务，每隔10ms会执行一次，然后会执行一系列的操作，那么我们就断点打在LongPollingRunnable的方法中，我们看到在控制台修改了配置之后，断点一路往下跑到cache.setContent()中，计算出的md5值已经不是原来的md5值，此时会更新CacheData，并且会进入checkListenerMd5()方法中，执行safeNotifyListener()方法，随后会执行listener.receiveConfigInfo()方法，通过断点跟踪找到了它的实现在NacosContextRefresher类中，位于com.alibaba.cloud.nacos.refresh包下，

```java
private void registerNacosListener(final String group, final String dataId) {

   Listener listener = listenerMap.computeIfAbsent(dataId, i -> new Listener() {
      @Override
      public void receiveConfigInfo(String configInfo) {
         refreshCountIncrement();
         String md5 = "";
         if (!StringUtils.isEmpty(configInfo)) {
            try {
               MessageDigest md = MessageDigest.getInstance("MD5");
               md5 = new BigInteger(1, md.digest(configInfo.getBytes("UTF-8")))
                     .toString(16);
            }
            catch (NoSuchAlgorithmException | UnsupportedEncodingException e) {
               log.warn("[Nacos] unable to get md5 for dataId: " + dataId, e);
            }
         }
         refreshHistory.add(dataId, md5);
         applicationContext.publishEvent(
               new RefreshEvent(this, null, "Refresh Nacos config"));
         if (log.isDebugEnabled()) {
            log.debug("Refresh Nacos config group " + group + ",dataId" + dataId);
         }
      }

      @Override
      public Executor getExecutor() {
         return null;
      }
   });

   try {
      configService.addListener(dataId, group, listener);
   }
   catch (NacosException e) {
      e.printStackTrace();
   }
}
```

​	可以看到，方法中首先根据dataId和新的configInfo计算出新的md5值，再把dataId和新的md5放到刷新的历史记录里去，随后会发布一个RefreshEvent刷新事件，那么这个事件会由谁来处理呢，继续断点跟踪，来到了RefreshEventListener类中的handle()方法，位于org.springframework.cloud.endpoint.event包中

```java
public void handle(RefreshEvent event) {
   if (this.ready.get()) { // don't handle events before app is ready
      log.debug("Event received " + event.getEventDesc());
      Set<String> keys = this.refresh.refresh();
      log.info("Refresh keys changed: " + keys);
   }
}
```

​	可以看到，之前控制台上的输出文字就是来自这个方法

```
2020-11-18 11:44:28.010  INFO 6376 --- [ost_8848-public] o.s.c.e.event.RefreshEventListener       : Refresh keys changed: [number]
```

## 五、总结

​	本文从实际应用出发，从什么是分布式配置中心开始，介绍了分布式配置中心的概念，SpringCloud Alibaba和nacos，接着又从官方示例项目入手，分析了nacos-config的客户端工作原理，希望大家阅读本文之后能有对nacos-config和分布式配置中心有所了解，有所收获，本人文笔水平有限，若有错误或不足，还望指正，谢谢大家
