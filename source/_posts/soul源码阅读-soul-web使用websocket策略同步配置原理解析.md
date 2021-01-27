---
title: soul源码阅读-soul-web使用websocket策略同步配置原理解析
date: 2021-01-27 13:03:33
tags:
- [网关]
- [soul]
- [Java]
categories: 
- [网关, soul]
- [Java]
---

## 前言

- 在之前的文章中说到了soul的http长轮询数据同步策略，这次说一下另一个跟http有点关系的同步策略websocket，websocket策略与长轮询相反，**它是admin主动向网关push数据**，在官网的文章中，也介绍了websocket的同步原理，结合官网的文章看一下websocket的整体同步流程。
<!--more-->


## 连接建立



在我们将网关同步策略配置为websocket后，启动项目，加载我们在网关配置的同步策略



```
soul :
    sync:
    websocket :
        urls: ws://localhost:9095/websocket # 如果是网关集群，这里可以配置多个，用','分割，比如ws://localhost:9095/websocket，ws://localhost:9096/websocket
```



之后会向目标地址建立websocket连接，在建立websocket连接的时候，会对每个连接设置30s间隔的断线重连，具体代码如下



```
public WebsocketSyncDataService(final WebsocketConfig websocketConfig,
                                    final PluginDataSubscriber pluginDataSubscriber,
                                    final List<MetaDataSubscriber> metaDataSubscribers,
                                    final List<AuthDataSubscriber> authDataSubscribers) {
            // 解析url，根据url的数量，创建对应的守护调度线程
        String[] urls = StringUtils.split(websocketConfig.getUrls(), ",");
        executor = new ScheduledThreadPoolExecutor(urls.length, SoulThreadFactory.create("websocket-connect", true));
        for (String url : urls) {
            try {
                clients.add(new SoulWebsocketClient(new URI(url), Objects.requireNonNull(pluginDataSubscriber), metaDataSubscribers, authDataSubscribers));
            } catch (URISyntaxException e) {
                log.error("websocket url({}) is error", url, e);
            }
        }
        try {
          // 对每个连接都设置同步阻塞方式，30s超时时间
            for (WebSocketClient client : clients) {
                boolean success = client.connectBlocking(3000, TimeUnit.MILLISECONDS);
                if (success) {
                    log.info("websocket connection is successful.....");
                } else {
                    log.error("websocket connection is error.....");
                }
              // 连接成功后，每30s检查一下心跳
                executor.scheduleAtFixedRate(() -> {
                    try {
                        if (client.isClosed()) {
                            boolean reconnectSuccess = client.reconnectBlocking();
                            if (reconnectSuccess) {
                                log.info("websocket reconnect is successful.....");
                            } else {
                                log.error("websocket reconnection is error.....");
                            }
                        }
                    } catch (InterruptedException e) {
                        log.error("websocket connect is error :{}", e.getMessage());
                    }
                }, 10, 30, TimeUnit.SECONDS);
            }
            /* client.setProxy(new Proxy(Proxy.Type.HTTP, new InetSocketAddress("proxyaddress", 80)));*/
        } catch (InterruptedException e) {
            log.info("websocket connection...exception....", e);
        }

    }
```



## 主动Push



- 此时web与admin的连接就建立完成了，只需要等待admin发生修改操作，将数据push到web更新本地缓存就好，我们来看看admin的push过程，以及web收到通知后的操作，在admin修改一个配置提交,定位到PluginServiceImpl#createOrUpdate方法。



```
public String createOrUpdate(final PluginDTO pluginDTO) {
        final String msg = checkData(pluginDTO);
        if (StringUtils.isNoneBlank(msg)) {
            return msg;
        }
        PluginDO pluginDO = PluginDO.buildPluginDO(pluginDTO);
        DataEventTypeEnum eventType = DataEventTypeEnum.CREATE;
        if (StringUtils.isBlank(pluginDTO.getId())) {
            pluginMapper.insertSelective(pluginDO);
        } else {
            eventType = DataEventTypeEnum.UPDATE;
            pluginMapper.updateSelective(pluginDO);
        }

        // publish change event.
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.PLUGIN, eventType,
                Collections.singletonList(PluginTransfer.INSTANCE.mapToData(pluginDO))));
        return StringUtils.EMPTY;
    }
```



在admin中，与其他的数据同步策略处理一样，都是通过Spring事件发布机制处理，看下最终通过DataChangedEventDispatcher定位到的方法



```
public void onPluginChanged(final List<PluginData> pluginDataList, final DataEventTypeEnum eventType) {
        WebsocketData<PluginData> websocketData =
                new WebsocketData<>(ConfigGroupEnum.PLUGIN.name(), eventType.name(), pluginDataList);
        WebsocketCollector.send(GsonUtils.getInstance().toJson(websocketData), eventType);
    }
```



通过listener定位到了websocketDataChangedListener中，之后通过WebsocketCollector#send方法发送数据，看看WebsocketCollector#send方法



```
public static void send(final String message, final DataEventTypeEnum type) {
        if (StringUtils.isNotBlank(message)) {
            if (DataEventTypeEnum.MYSELF == type) {
                try {
                    Session session = (Session) ThreadLocalUtil.get(SESSION_KEY);
                    if (session != null) {
                        session.getBasicRemote().sendText(message);
                    }
                } catch (IOException e) {
                    log.error("websocket send result is exception: ", e);
                }
                return;
            }
            for (Session session : SESSION_SET) {
                try {
                    session.getBasicRemote().sendText(message);
                } catch (IOException e) {
                    log.error("websocket send result is exception: ", e);
                }
            }
        }
    }
```



- 在admin通过websocket发送之后，SoulWebsocketClient#onMessage接收数据最终在handleResult方法进行处理



```
private void handleResult(final String result) {
  // json反序列化之后，获取到configGroup和eventType事件类型最终进行处理
        WebsocketData websocketData = GsonUtils.getInstance().fromJson(result, WebsocketData.class);
        ConfigGroupEnum groupEnum = ConfigGroupEnum.acquireByName(websocketData.getGroupType());
        String eventType = websocketData.getEventType();
        String json = GsonUtils.getInstance().toJson(websocketData.getData());
        websocketDataHandler.executor(groupEnum, json, eventType);
    }
// WebsocketDataHandler#executor
public void executor(final ConfigGroupEnum type, final String json, final String eventType) {
        ENUM_MAP.get(type).handle(json, eventType);
    }
```



- 然后定位到了AbstractDataHandler中的handle方法（又是一个运用了模版模式的地方）



```
public void handle(final String json, final String eventType) {
  // 解析json后根据不同的具体实现类和eventType，定位到具体的实现类中进行执行
        List<T> dataList = convert(json);
        if (CollectionUtils.isNotEmpty(dataList)) {
            DataEventTypeEnum eventTypeEnum = DataEventTypeEnum.acquireByName(eventType);
            switch (eventTypeEnum) {
                case REFRESH:
                case MYSELF:
                    doRefresh(dataList);
                    break;
                case UPDATE:
                case CREATE:
                    doUpdate(dataList);
                    break;
                case DELETE:
                    doDelete(dataList);
                    break;
                default:
                    break;
            }
        }
    }
```



- 我们修改插件，因此是Plugin类型，最终定位到了PluginDataHandler#doUpdate方法



```
protected void doUpdate(final List<PluginData> dataList) {
  // 就一行代码，依次订阅，而具体查看这个onSubscribe方法时，PluginDataSubscriber就只有一个具体实现CommonPluginDataSubscriber，最终会定位到CommonPluginDataSubscriber的具体实现中
        dataList.forEach(pluginDataSubscriber::onSubscribe);
    }

//CommonPluginDataSubscriber
//逻辑上处理也相对简单，判断事件类型，更新对应类型的缓存
//每个分支下面都用到了BaseDataCache，见名知意，网关里面各种插件数据类型的数据都缓存在这里了。
//之后的handler.xxx方法，其实就是对应的数据类型的处理器了，接口里面大多使用了默认实现，具体处理器实现接口根据需要进行重写
private <T> void subscribeDataHandler(final T classData, final DataEventTypeEnum dataType) {
        Optional.ofNullable(classData).ifPresent(data -> {
            if (data instanceof PluginData) {
                PluginData pluginData = (PluginData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    BaseDataCache.getInstance().cachePluginData(pluginData);
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.handlerPlugin(pluginData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    BaseDataCache.getInstance().removePluginData(pluginData);
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.removePlugin(pluginData));
                }
            } else if (data instanceof SelectorData) {
                SelectorData selectorData = (SelectorData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    BaseDataCache.getInstance().cacheSelectData(selectorData);
                    Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.handlerSelector(selectorData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    BaseDataCache.getInstance().removeSelectData(selectorData);
                    Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.removeSelector(selectorData));
                }
            } else if (data instanceof RuleData) {
                RuleData ruleData = (RuleData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    BaseDataCache.getInstance().cacheRuleData(ruleData);
                    Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.handlerRule(ruleData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    BaseDataCache.getInstance().removeRuleData(ruleData);
                    Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.removeRule(ruleData));
                }
            }
        });
    }
```



```
// 比如divide插件，就只充血了3个方法，其余的都是用接口默认方法。
public class DividePluginDataHandler implements PluginDataHandler {
    
    @Override
    public void handlerSelector(final SelectorData selectorData) {
        UpstreamCacheManager.getInstance().submit(selectorData);
    }
    
    @Override
    public void removeSelector(final SelectorData selectorData) {
        UpstreamCacheManager.getInstance().removeByKey(selectorData.getId());
    }
    
    @Override
    public String pluginNamed() {
        return PluginEnum.DIVIDE.getName();
    }
}
```



## 结尾



- 到这里，websocket的数据同步策略一次具体的同步流程就完成了，soul在进行设计的时候进行了高度抽象，大部分的代码都做到了复用，具体的功能特性只需进行少量定制实现就好。最后理一下websocket数据同步策略的流程：

![img](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/f1e5e13ea4ed8f7232b76e0219965c75-20210127123427679-20210127123435841.svg)