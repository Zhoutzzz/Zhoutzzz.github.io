---
title: soul源码阅读-soul-web使用http长轮询策略同步配置原理解析
date: 2021-01-27 13:04:24
tags:
- [网关]
- [soul]
- [Java]
categories: 
- [网关, soul]
- [Java]
---

## 简介


  - 在参考soul官网研究soul的数据同步策略的时候发现，soul目前支持的数据同步策略有zk，websocket和http长轮询，而zk和websocke是主动push的策略，在admin进行配置修改的时候才会触发，长轮询作为pull策略，通过soul-web主动向admin发起数据同步策略，唯一的pull策略引发了兴趣，研究下soul如何实现的。
  - 在官网的介绍里说了http长轮询的执行流程，web网关，会定时向admin发起数据同步的长轮询请求，而admin则在接受到请求后，使用servlet3.0的异步特性，先将请求放入一个阻塞队列ArrayBlockingQueue中保存，发生了数据变更时，会将队列中的所有任务出队依次处理并响应，如果超时，则将队列中的头部元素取出进行处理然后响应。
  - 既然是数据同步的策略，那具体的实现就应该在soul-sync-data-center中，http长轮询就应该对应soul-sync-data-http模块（soul的项目结构本身就设计的很好，并且命名规范，按照功能进行模块拆分，具体的功能点可以在对应的模块中找到，在加上阅读源码时的‘假设性原则’，定位到核心代码处更是分分钟的事情，再不济也能猜到入口，一步步debug）
  <!--more-->

  ![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1611320552588-c82f9048-4f15-4579-bb16-d0a9765e6e86-20210127123543859.png)

  ## soul-web长轮询初始化

  - 目录下的HttpSyncDataService，就是长轮询的发起点，refresh包中，则是对应的各个类型的数据具体的更新实现，在收到admin的响应后判断是否执行刷新。在HttpSyncDataService中，有一个属性证明了这就是长轮询同步的实现

  ![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1611321720728-f1164b8d-b1b7-4fc8-b009-d619805e40c2-20210127125700577.png)

  - 默认http连接超时时间10s，下面是初始化相关的方法

  ```
  public HttpSyncDataService(final HttpConfig httpConfig, final PluginDataSubscriber pluginDataSubscriber,
                                 final List<MetaDataSubscriber> metaDataSubscribers, final List<AuthDataSubscriber> authDataSubscribers) {
          this.factory = new DataRefreshFactory(pluginDataSubscriber, metaDataSubscribers, authDataSubscribers);
          this.httpConfig = httpConfig;
          this.serverList = Lists.newArrayList(Splitter.on(",").split(httpConfig.getUrl()));
          this.httpClient = createRestTemplate();
          this.start();
      }
      // 定义restTemplate
      private RestTemplate createRestTemplate() {
          OkHttp3ClientHttpRequestFactory factory = new OkHttp3ClientHttpRequestFactory();
        // 10s连接超时
          factory.setConnectTimeout((int) this.connectionTimeout.toMillis());
        // 90s读取超时
          factory.setReadTimeout((int) HttpConstants.CLIENT_POLLING_READ_TIMEOUT);
          return new RestTemplate(factory);
      }
  
      private void start() {
          // It could be initialized multiple times, so you need to control that.
        // 使用cas设置为true，表示需要进行一次同步
          if (RUNNING.compareAndSet(false, true)) {
              // fetch all group configs. 先获取所有组的配置，更新缓存
              this.fetchGroupConfig(ConfigGroupEnum.values());
              int threadSize = serverList.size();
            // 初始化执行长轮询任务的线程池
              this.executor = new ThreadPoolExecutor(threadSize, threadSize, 60L, TimeUnit.SECONDS,
                      new LinkedBlockingQueue<>(),
                      SoulThreadFactory.create("http-long-polling", true));
              // start long polling, each server creates a thread to listen for changes.
            // 发起长轮询
              this.serverList.forEach(server -> this.executor.execute(new HttpLongPollingTask(server)));
          } else {
              log.info("soul http long polling was started, executor=[{}]", executor);
          }
      }
  ```

  ## 定时请求任务

  - 初始化方法里，参数设定完成后启动长轮询任务，具体执行就在HttpLongPollingTask中了

  ```
  class HttpLongPollingTask implements Runnable {
  
          private String server;
  
          private final int retryTimes = 3;
  
          HttpLongPollingTask(final String server) {
              this.server = server;
          }
  
          @Override
          public void run() {
            // 在初始化时启动的时候，就将状态设置为了true，无限循环调用，如果遇到异常，重试3次
              while (RUNNING.get()) {
                  for (int time = 1; time <= retryTimes; time++) {
                      try {
                        // 发起长轮询请求
                          doLongPolling(server);
                      } catch (Exception e) {
                          // print warnning log.
                          if (time < retryTimes) {
                              log.warn("Long polling failed, tried {} times, {} times left, will be suspended for a while! {}",
                                      time, retryTimes - time, e.getMessage());
                              ThreadUtils.sleep(TimeUnit.SECONDS, 5);
                              continue;
                          }
                          // print error, then suspended for a while.
                          log.error("Long polling failed, try again after 5 minutes!", e);
                          ThreadUtils.sleep(TimeUnit.MINUTES, 5);
                      }
                  }
              }
              log.warn("Stop http long polling.");
          }
      }
  private void doLongPolling(final String server) {
          MultiValueMap<String, String> params = new LinkedMultiValueMap<>(8);
    // 因为soul中每个类型的数据，都有对应的md5进行标识，使用md5与类型关联，如果md5发生变动，表示需要进行同步
          for (ConfigGroupEnum group : ConfigGroupEnum.values()) {
              ConfigData<?> cacheConfig = factory.cacheConfigData(group);
              String value = String.join(",", cacheConfig.getMd5(), String.valueOf(cacheConfig.getLastModifyTime()));
              params.put(group.name(), Lists.newArrayList(value));
          }
          HttpHeaders headers = new HttpHeaders();
          headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
          HttpEntity httpEntity = new HttpEntity(params, headers);
    
              // 具体admin监听接口path
          String listenerUrl = server + "/configs/listener";
          log.debug("request listener configs: [{}]", listenerUrl);
          JsonArray groupJson = null;
          try {
            
            // 向admin发起请求，admin是延迟响应，不会立刻响应，不然task无限请求服务器顶不住的
              String json = this.httpClient.postForEntity(listenerUrl, httpEntity, String.class).getBody();
              log.debug("listener result: [{}]", json);
              groupJson = GSON.fromJson(json, JsonObject.class).getAsJsonArray("data");
          } catch (RestClientException e) {
              String message = String.format("listener configs fail, server:[%s], %s", server, e.getMessage());
              throw new SoulException(message, e);
          }
          if (groupJson != null) {
              // fetch group configuration async.
            //admin响应之后，根据返回的结果判断是否需要进行配置更新，配置更新是异步进行
              ConfigGroupEnum[] changedGroups = GSON.fromJson(groupJson, ConfigGroupEnum[].class);
              if (ArrayUtils.isNotEmpty(changedGroups)) {
                  log.info("Group config changed: {}", Arrays.toString(changedGroups));
                  this.doFetchGroupConfig(server, changedGroups);
              }
          }
      }
  ```

  ## 配置更新

  - 具体的配置更新方法

  ```
  private void doFetchGroupConfig(final String server, final ConfigGroupEnum... groups) {
          StringBuilder params = new StringBuilder();
          for (ConfigGroupEnum groupKey : groups) {
              params.append("groupKeys").append("=").append(groupKey.name()).append("&");
          }
  
              // admin获取配置的path
          String url = server + "/configs/fetch?" + StringUtils.removeEnd(params.toString(), "&");
          log.info("request configs: [{}]", url);
          String json = null;
          try {
              json = this.httpClient.getForObject(url, String.class);
          } catch (RestClientException e) {
              String message = String.format("fetch config fail from server[%s], %s", url, e.getMessage());
              log.warn(message);
              throw new SoulException(message, e);
          }
          // update local cache
          boolean updated = this.updateCacheWithJson(json);
          if (updated) {
              log.info("get latest configs: [{}]", json);
              return;
          }
          // not updated. it is likely that the current config server has not been updated yet. wait a moment.
    // 这里等待30s再次监听，不等待也可能导致一直向admin请求，耗空资源
          log.info("The config of the server[{}] has not been updated or is out of date. Wait for 30s to listen for changes again.", server);
          ThreadUtils.sleep(TimeUnit.SECONDS, 30);
      }
  
      /**
       * update local cache.
       * @param json the response from config server.
       * @return true: the local cache was updated. false: not updated.
       */
      private boolean updateCacheWithJson(final String json) {
          JsonObject jsonObject = GSON.fromJson(json, JsonObject.class);
          JsonObject data = jsonObject.getAsJsonObject("data");
          // if the config cache will be updated?
          return factory.executor(data);
      }
  // 具体的executor
  public boolean executor(final JsonObject data) {
    设置默认值false，否则调用这个方法如果返回true，也会引发资源耗尽的问题
          final boolean[] success = {false};
          ENUM_MAP.values().parallelStream().forEach(dataRefresh -> success[0] = dataRefresh.refresh(data));
          return success[0];
      }
  ```

  ## soul-admin接收

  - 到这里，web的长轮询具体逻辑就完成了，接下来就是看请求发送到admin之后admin怎么处理了

  ```
  /**
   * This Controller only when HttpLongPollingDataChangedListener exist, will take effect.
   这个controller只有当长轮询策略生效的时候才会加载。
   *
   * @author huangxiaofeng
   * @author xiaoyu
   */
  @ConditionalOnBean(HttpLongPollingDataChangedListener.class)
  @RestController
  @RequestMapping("/configs")
  @Slf4j
  public class ConfigController {
  ```

  - 看看接口里面的2个方法,/fetch和/listener正好与前面代码中配置的路径对应上了

  ```
  @GetMapping("/fetch")
      public SoulAdminResult fetchConfigs(@NotNull final String[] groupKeys) {
          Map<String, ConfigData<?>> result = Maps.newHashMap();
          for (String groupKey : groupKeys) {
              ConfigData<?> data = longPollingListener.fetchConfig(ConfigGroupEnum.valueOf(groupKey));
              result.put(groupKey, data);
          }
          return SoulAdminResult.success(SoulResultMessage.SUCCESS, result);
      }
  
      /**
       * Listener.
       *
       * @param request  the request
       * @param response the response
       */
      @PostMapping(value = "/listener")
      public void listener(final HttpServletRequest request, final HttpServletResponse response) {
          longPollingListener.doLongPolling(request, response);
      }
  ```

  ## 任务处理

  - 我们看看对应的fetchConfig和doLongPolling方法具体内容

  ```
  // If the configuration data changes, the group information for the change is immediately responded.
      // Otherwise, the client's request thread is blocked until any data changes or the specified timeout is reached.
      
      //这个方法头上注解说明了，如果配置数据有更改，对应类型的信息会被立刻包装为response响应给web发起的http请求，否则客户端的请求线程会阻塞知道数据发生更改或者超时，源码内部将请求的超时时间设为60s，也就是说，如果我们在admin进行配置修改，会立马对web发出的长轮询作出响应，不然就这边等60s，这也就是web那根据atomicBoolean的状态不会无限请求的原因。
  public void doLongPolling(final HttpServletRequest request, final HttpServletResponse response) {
  
          // compare group md5 实际判断方式是比较类型的md5，只要有任意类型被修改了，这个list都不会为空，就会立刻响应
          List<ConfigGroupEnum> changedGroup = compareChangedGroup(request);
          String clientIp = getRemoteIp(request);
  
          // response immediately.
          if (CollectionUtils.isNotEmpty(changedGroup)) {
              this.generateResponse(response, changedGroup);
              log.info("send response with the changed group, ip={}, group={}", clientIp, changedGroup);
              return;
          }
  
          // listen for configuration changed. 使用异步方式进行监听
          final AsyncContext asyncContext = request.startAsync();
  
          // AsyncContext.settimeout() does not timeout properly, so you have to control it yourself
          asyncContext.setTimeout(0L);
  
          // block client's thread.
          scheduler.execute(new LongPollingClient(asyncContext, clientIp, 
          // 具体的超时枚举
          HttpConstants.SERVER_MAX_HOLD_TIMEOUT));
      }
  ```

  - 最终将web的请求，放入调度线程池异步执行，在分析@SoulSpringMvcClient注解注册流程的时候也提到过这个实现了runnable的任务类，具体的任务方法如下

  ```
  public void run() {
    // 定义调度线程池调度执行时的具体方法，在调度线程池执行的时候，创建新的runnable来执行，将阻塞队列中的头部任务出队，然后比较数据，发送响应
              this.asyncTimeoutFuture = scheduler.schedule(() -> {
                  clients.remove(LongPollingClient.this);
                  List<ConfigGroupEnum> changedGroups = compareChangedGroup((HttpServletRequest) asyncContext.getRequest());
                  sendResponse(changedGroups);
              }, timeoutTime, TimeUnit.MILLISECONDS);
    // 直接将当前任务加入队列。
              clients.add(this);
          }
  ```

  - LongPollingClient的run方法在发送响应的时候，会根据changedGroup来生成响应数据，如果这个changedGroup元素为0，生成的响应数据是“[{}]”，如果有数据，就会将对应数据解析为json响应，之前说doLongPolling方法的时候，就解释了对于响应的数据会做何种处理，如果不需要更新就重新发起90秒的任务，如果需要更新，这时候web就会去根据访问/listener接口得到的具体的group数据，调用/fetch获取对应的配置进行更新。

  ```
  // 根据枚举选择对应的group从缓存中拿出数据返回。
  
  public ConfigData<?> fetchConfig(final ConfigGroupEnum groupKey) {
          ConfigDataCache config = CACHE.get(groupKey.name());
          switch (groupKey) {
              case APP_AUTH:
                  List<AppAuthData> appAuthList = GsonUtils.getGson().fromJson(config.getJson(), new TypeToken<List<AppAuthData>>() {
                  }.getType());
                  return new ConfigData<>(config.getMd5(), config.getLastModifyTime(), appAuthList);
              case PLUGIN:
                  List<PluginData> pluginList = GsonUtils.getGson().fromJson(config.getJson(), new TypeToken<List<PluginData>>() {
                  }.getType());
                  return new ConfigData<>(config.getMd5(), config.getLastModifyTime(), pluginList);
              case RULE:
                  List<RuleData> ruleList = GsonUtils.getGson().fromJson(config.getJson(), new TypeToken<List<RuleData>>() {
                  }.getType());
                  return new ConfigData<>(config.getMd5(), config.getLastModifyTime(), ruleList);
              case SELECTOR:
                  List<SelectorData> selectorList = GsonUtils.getGson().fromJson(config.getJson(), new TypeToken<List<SelectorData>>() {
                  }.getType());
                  return new ConfigData<>(config.getMd5(), config.getLastModifyTime(), selectorList);
              case META_DATA:
                  List<MetaData> metaList = GsonUtils.getGson().fromJson(config.getJson(), new TypeToken<List<MetaData>>() {
                  }.getType());
                  return new ConfigData<>(config.getMd5(), config.getLastModifyTime(), metaList);
              default:
                  throw new IllegalStateException("Unexpected groupKey: " + groupKey);
          }
      }
  ```

  - 感觉这怎么跟官网上面说的有点不太符合，数据发生变动，为什么就请求接口从缓存拿数据了，不是会把前面的任务队列中的任务全部出队然后处理并响应么。因为当web请求过来之后被保存到了队列中，配置在admin的控制台修改之后，会触发监听，而监听会根据具体的配置修改的类型将修改事件分发到不同的事件处理器中，在事件处理器中会更新admin的缓存数据，同时让任务队列执行DataChangeTask任务，在DataChangeTask中具体的方法如下，admin修改配置后具体的数据同步逻辑，在讲解@SoulSpringMvcClient的文章后半部分中已经说明。

  ```
  public void run() {
    // 将队列中的任务循环取出后挨个发送它们对应的响应。
              for (Iterator<LongPollingClient> iter = clients.iterator(); iter.hasNext();) {
                  LongPollingClient client = iter.next();
                  iter.remove();
                  client.sendResponse(Collections.singletonList(groupKey));
                  log.info("send response with the changed group,ip={}, group={}, changeTime={}", client.ip, groupKey, changeTime);
              }
          }
  ```

  - 到此这个http长轮询策略具体流程就和官方文章中介绍的一致了，最后画个图理一理它的整个同步流程![未命名文件.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1611330594826-6808f987-e16b-416f-a9cc-d904763d3298-20210127125714815.png)