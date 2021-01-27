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

- Divide插件，作为soul进行http协议请求处理的核心插件，实际流程，插件提供的负载均衡，路径匹配，服务探活具体实现
<!--more-->
```java
protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
    final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
    assert soulContext != null;
    final DivideRuleHandle ruleHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), DivideRuleHandle.class);
  //  确保请求的上游服务存在，里面会探活
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

- `UpstreamCacheManager`，divide插件的探活使用的具体实现，使用到这个缓存管理器的地方在`DividePluginDataHandler`中，处理插件数据的时候会去探活。

```java
private UpstreamCacheManager() {
  初始化探活调度器
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

