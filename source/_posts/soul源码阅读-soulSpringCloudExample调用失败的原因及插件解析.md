---
title: soul源码阅读-soulSpringCloudExample调用失败的原因及插件解析
date: 2021-01-27 13:04:50
tags:
- [网关]
- [soul]
- [Java]
categories: 
- [网关, soul]
- [Java]
---
## 原因

- 在一开始使用spring cloud插件的时候，调用网关没有得到请求，一直返回：Can not find url, please check your configuration!
- 先说原因，是因为在bootstrap中，没有打开spring cloud的相关依赖导致的，具体分析如下。
<!--more-->
## 分析

- 首先，我们通过网关访问我们的项目，我们知道soul是集成插件达到各种功能的，那么，我们就不用去找soul的入口，可以先去找插件，
- 插件的类依赖关系就像下面这样

![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1611153559988-4e631302-ebde-423a-a947-fd6118f16eef-20210127125336545.png)

- 可以看到，顶层是接口，接口的唯一直接实现，是这个抽象类AbstractSoulPlugin，而SpringCloudPlugin继承抽象类做具体的实现。
- 我们可以看下，抽象类里面都是什么东东。

![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1611153889175-59a89b58-8cf4-4a30-91bb-cefa529d7387-20210127125351681.png)

- 类里面方法太多了，就贴最核心的2个方法吧，一个protected的抽象方法doExecute，一个public的方法execute，**注意看抽象方法的注释，说明这是模版方法，子类继承的时候在里面实现自己的逻辑，非常标准的模版模式运用（**模版模式：简单来说，就是对外提供一个抽象方法，交给子类实现，这个抽象方法的调用，则在抽象类的骨架方法里面调用，上图的execute方法，就是骨架方法，还不清楚，就去查查资料吧**）**。
- 其实还有2个直接实现接口的类WebClientPlugin，GlobalPlugin

![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1611153724359-cdc744df-be5b-469f-9f17-607d9d62709c-20210127125417589.png)![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1611159700407-fa59d390-8368-4d1a-a2a1-a746eb4e6ff7-20210127125411346.png)

- 先说WebClientPlugin，为啥不继承抽象类，要单独实现呢，个人分析是因为这其实是soul在进行http调用时的核心插件，所有的http调用，都是由这个插件来完成，是soul的默认插件，也许细心的你会发现，在soul-admin中的web控制台中，这个插件不在插件列表里，你不能操作它，而继承了抽象类的插件们，是对客户端提供的可选插件，虽然所有的插件，都在一个插件链里面，但是他们的价值不同，所以，在具体实现上区别开了。
- 而GlobalPlugin插件，见名知意，是整个soul插件的基础，看看GlobalPlugin的关键方法。

```
    @Override
// 注意，接收的是一个插件链参数，在全局插件中，调用插件链的执行方法，执行每一个插件
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        final ServerHttpRequest request = exchange.getRequest();
        final HttpHeaders headers = request.getHeaders();
        final String upgrade = headers.getFirst("Upgrade");
        SoulContext soulContext;
        if (StringUtils.isBlank(upgrade) || !"websocket".equals(upgrade)) {
            soulContext = builder.build(exchange);
        } else {
            final MultiValueMap<String, String> queryParams = request.getQueryParams();
            soulContext = transformMap(queryParams);
        }
        exchange.getAttributes().put(Constants.CONTEXT, soulContext);
        return chain.execute(exchange);
    }
    
    @Override
    public int getOrder() {
        // 后面讲到插件排序就有用了，永远是插件链的第一个插件。
        return 0;
    }
```



- 来看看execute这个骨架方法具体都干什么

```
// ServerWebExchange是webflux的请求处理器
public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        String pluginName = named();  // 获取插件的名称
        final PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName); //根据名称从缓存拿插件元数据
        if (pluginData != null && pluginData.getEnabled()) {
            //拿到插件的选择器配置根据URL进行匹配
            final Collection<SelectorData> selectors = BaseDataCache.getInstance().obtainSelectorData(pluginName);
            if (CollectionUtils.isEmpty(selectors)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            final SelectorData selectorData = matchSelector(exchange, selectors);
            if (Objects.isNull(selectorData)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            selectorLog(selectorData, pluginName);
            //拿到插件的规则配置根据URL进行规则匹配
            final List<RuleData> rules = BaseDataCache.getInstance().obtainRuleData(selectorData.getId());
            if (CollectionUtils.isEmpty(rules)) {
                return handleRuleIsNull(pluginName, exchange, chain);
            }
            RuleData rule;
            if (selectorData.getType() == SelectorTypeEnum.FULL_FLOW.getCode()) {
                //get last
                rule = rules.get(rules.size() - 1);
            } else {
                rule = matchRule(exchange, rules);
            }
            if (Objects.isNull(rule)) {
                return handleRuleIsNull(pluginName, exchange, chain);
            }
            ruleLog(rule, pluginName);
            return doExecute(exchange, chain, selectorData, rule);
        }
        return chain.execute(exchange)
    }
```

- 既然说了插件链，那就应该看看，插件链在哪，怎么生成的，链的顺序是怎样的

- - 在模版方法处debug，逆向寻找，最终找到了插件链定义的地方：**SoulWebHandler**![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1611156130907-01526587-ec64-4bf3-ba5d-a0dd7fc37461-20210127125508594.png)
  - 查看构造方法的调用，找到了插件链生成的地方：SoulConfiguration**
    **

![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1611156311825-1c6ece72-7f0b-4f4f-bd4e-2592a40ddd46-20210127125521865.png)

- - 通过Spring的bean属性注入，来构造的插件链，Spring在执行该方法的时候，会根据方法参数，将所有实现了SoulPlugin接口并且委托给Spring管理的bean都封装后传入。
  - **注意看，插件链在构造的时候，是排了序的**，为什么要排序，要归根于插件具体的作用不同，有的插件是初始化的，有的是参数处理的，有的是实际执行的，举个例子，刚刚说了，soul中进行http调用实际上用的是wenClientPlugin插件来执行的，那么，那些要使用http调用的插件，比如divide插件，不是重复了么，那我们来看看divide插件里面，干了什么。

```
protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        // 获取soul上下文，拿到插件的规则处理器    
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        final DivideRuleHandle ruleHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), DivideRuleHandle.class);
        final List<DivideUpstream> upstreamList = UpstreamCacheManager.getInstance().findUpstreamListBySelectorId(selector.getId());
        if (CollectionUtils.isEmpty(upstreamList)) {
            log.error("divide upstream configuration error： {}", rule.toString());
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        final String ip = Objects.requireNonNull(exchange.getRequest().getRemoteAddress()).getAddress().getHostAddress();
        DivideUpstream divideUpstream = LoadBalanceUtils.selector(upstreamList, ruleHandle.getLoadBalance(), ip);
        if (Objects.isNull(divideUpstream)) {
            log.error("divide has no upstream");
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        // set the http url 注释也写了，设置http的url和超时配置。
        String domain = buildDomain(divideUpstream);
        String realURL = buildRealURL(domain, soulContext, exchange);
        exchange.getAttributes().put(Constants.HTTP_URL, realURL);
        // set the http timeout
        exchange.getAttributes().put(Constants.HTTP_TIME_OUT, ruleHandle.getTimeout());
        exchange.getAttributes().put(Constants.HTTP_RETRY, ruleHandle.getRetry());
        return chain.execute(exchange);
    }
```

- - 可以看到，就是设置了下http的url跟一些相关参数，前面我们说了，soul插件链的处理，使用了模版模式做了一层抽象设计，http的调用，用的是souWebClient插件，这就要看看souWebClient的模版方法写的啥了。

```
@Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        // 拿到url和相关参数
        String urlPath = exchange.getAttribute(Constants.HTTP_URL);
        if (StringUtils.isEmpty(urlPath)) {
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        long timeout = (long) Optional.ofNullable(exchange.getAttribute(Constants.HTTP_TIME_OUT)).orElse(3000L);
        int retryTimes = (int) Optional.ofNullable(exchange.getAttribute(Constants.HTTP_RETRY)).orElse(0);
        log.info("The request urlPath is {}, retryTimes is {}", urlPath, retryTimes);
        //封装成http请求发起调用
        HttpMethod method = HttpMethod.valueOf(exchange.getRequest().getMethodValue());
        WebClient.RequestBodySpec requestBodySpec = webClient.method(method).uri(urlPath);
        return handleRequestBody(requestBodySpec, exchange, timeout, retryTimes, chain);
    }
```

- - 所以，已经可以肯定soul中需要使用http进行调用的可选插件，都只是进行了http参数的设置，通过soulContext进行传递，交给webClient插件进行调用。也就能说通，为什么插件链生成的时候要排序了，如果不进行排序，webClient插件链在其他插件之前被执行了，那就GG了，类似的，基于dubbo， sofa协议的rpc调用，也会在实际执行之前，先进行请求的构建，只是具体实现稍有不同。具体插件们的顺序，**在****PluginEnum中定义了。**
  - **还有一个插件与WebClientPlugin一样比较特殊，GlobalPlugin，它是初始化插件**
  - 前面说了插件链的定义、生成，插件链的顺序，也说了具体的插件是通过Spring注入的，那么这些具体的插件bean在哪里交给Spring托管的呢

![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1611156560520-fe191cdf-7777-4aaa-8e36-9cd445cf43bc-20210127125536221.png)![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1611156612395-bc26793d-1d09-4308-b6d8-0625c634446c-20210127125544209.png)

**每一个插件，都提供了对应的starter来进行插件的实例化，从而可以在构造soulWebHandler时拿到插件。**

## 总结

- 至此，http插件的一个调用过程就说完了，原因就如一开始所说，spring cloud也是通过http来进行调用，pom没有加入Spring cloud相关的依赖，导致soul在创建webHandler的时候，没有加载到spring cloud插件，也就没有spring cloud相关的配置，自然也就调用不到实际的url了。
- 最后是插件调用链的图。

![img](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1611161979343-279f3241-95c3-4f8d-bda8-1f99e073b8ba-20210127125554348.jpeg)