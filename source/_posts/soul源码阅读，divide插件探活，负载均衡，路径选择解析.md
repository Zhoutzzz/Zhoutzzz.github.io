---
title: soul源码阅读，divide插件探活，负载均衡，路径选择解析
date: 2021-01-27 13:04:36
tags:
- [网关]
- [soul]
- [Java]
categories: 
- [网关, soul]
- [Java]
---

- Divide插件，作为soul进行http协议请求处理的核心插件，本文介绍divide具体工作流程，插件提供的负载均衡，服务探活具体实现，下面是插件的主体逻辑，网关在接收到客户端的请求时divide插件干的活全在这里。

```java
protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
    final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
    assert soulContext != null;
    final DivideRuleHandle ruleHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), DivideRuleHandle.class);
  //  确保请求的上游服务存在
    final List<DivideUpstream> upstreamList = UpstreamCacheManager.getInstance().findUpstreamListBySelectorId(selector.getId());
    if (CollectionUtils.isEmpty(upstreamList)) {
        log.error("divide upstream configuration error： {}", rule.toString());
        Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
        return WebFluxResultUtils.result(exchange, error);
    }
    final String ip = Objects.requireNonNull(exchange.getRequest().getRemoteAddress()).getAddress().getHostAddress();
  // 负载均衡，确定实际的上游后端服务地址
    DivideUpstream divideUpstream = LoadBalanceUtils.selector(upstreamList, ruleHandle.getLoadBalance(), ip);
    if (Objects.isNull(divideUpstream)) {
        log.error("divide has no upstream");
        Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
        return WebFluxResultUtils.result(exchange, error);
    }
    // set the http url
    String domain = buildDomain(divideUpstream);
    String realURL = buildRealURL(domain, soulContext, exchange);
    exchange.getAttributes().put(Constants.HTTP_URL, realURL);
    // set the http timeout
    exchange.getAttributes().put(Constants.HTTP_TIME_OUT, ruleHandle.getTimeout());
    exchange.getAttributes().put(Constants.HTTP_RETRY, ruleHandle.getRetry());
    return chain.execute(exchange);
}
```

- 之前文章说了关于http请求，实际的执行请求转发插件是`webClientPlugin`，其他的比如divide，springCloud插件，都是对当前http请求进行加工处理，处理完成交给`webClientP lugin`发送。
- `UpstreamCacheManager`是上游后端服务的缓存，divide插件处理过程中要确保此次请求的服务是存在的，才能保证后面的转发插件能正确转发请求到实际服务，因此就需要本地保存目前真实存在的后端服务，不影响性能的情况下还能确保请求转发的正确性，同时还需要发送心跳检查后端服务健康情况。

```java
private UpstreamCacheManager() {
  // 初始化探活调度器
        boolean check = Boolean.parseBoolean(System.getProperty("soul.upstream.check", "false"));
        if (check) {
            new ScheduledThreadPoolExecutor(1, SoulThreadFactory.create("scheduled-upstream-task", false))
                    .scheduleWithFixedDelay(this::scheduled,
                            30, Integer.parseInt(System.getProperty("soul.upstream.scheduledTime", "30")), TimeUnit.SECONDS);
        }
    }
	// 根据选择器，选择是否添加到上游服务缓存
    public void submit(final SelectorData selectorData) {
        final List<DivideUpstream> upstreamList = GsonUtils.getInstance().fromList(selectorData.getHandle(), DivideUpstream.class);
        if (null != upstreamList && upstreamList.size() > 0) {
            UPSTREAM_MAP.put(selectorData.getId(), upstreamList);
            UPSTREAM_MAP_TEMP.put(selectorData.getId(), upstreamList);
        } else {
            UPSTREAM_MAP.remove(selectorData.getId());
            UPSTREAM_MAP_TEMP.remove(selectorData.getId());
        }
    }

// 在进行调度的时候去检查服务是否可用
    private void scheduled() {
        if (UPSTREAM_MAP.size() > 0) {
            UPSTREAM_MAP.forEach((k, v) -> {
                List<DivideUpstream> result = check(v);
                if (result.size() > 0) {
                    UPSTREAM_MAP_TEMP.put(k, result);
                } else {
                    UPSTREAM_MAP_TEMP.remove(k);
                }
            });
        }
    }


    private List<DivideUpstream> check(final List<DivideUpstream> upstreamList) {
        List<DivideUpstream> resultList = Lists.newArrayListWithCapacity(upstreamList.size());
        for (DivideUpstream divideUpstream : upstreamList) {
          // 通过socket连接检查上游后端服务
            final boolean pass = UpstreamCheckUtils.checkUrl(divideUpstream.getUpstreamUrl());
            if (pass) {
                if (!divideUpstream.isStatus()) {
                    divideUpstream.setTimestamp(System.currentTimeMillis());
                    divideUpstream.setStatus(true);
                    log.info("UpstreamCacheManager detect success the url: {}, host: {} ", divideUpstream.getUpstreamUrl(), divideUpstream.getUpstreamHost());
                }
                resultList.add(divideUpstream);
            } else {
                divideUpstream.setStatus(false);
                log.error("check the url={} is fail ", divideUpstream.getUpstreamUrl());
            }
        }
        return resultList;

    }
```

- 这个缓存管理器里面，有2个缓存，一个`UPSTREAM_MAP`，一个`UPSTREAM_MAP_TEMP`，后者是内部实时更新，对外提供健康服务地址的缓存，前者是作为一致性检查的缓存，在进行调度的时候，根据`UPSTREAM_MAP`的内容，会对`UPSTREAM_MAP_TEMP`进行更新，保证每次心跳检测时两个缓存内容一致。
- 对缓存地址的添加则是在`DividePluginDataHandler`中操作的，当admin更新了后端服务地址时，通过发布订阅机制，本地的缓存会收到消息进行更新。