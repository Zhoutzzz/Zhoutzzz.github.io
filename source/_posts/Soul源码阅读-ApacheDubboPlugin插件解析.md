---
title: soul源码阅读-ApacheDubboPlugin插件解析
date: 2021-01-27 13:04:36
tags:
- [网关]
- [soul]
- [Java]
categories: 
- [网关, soul]
- [Java]
---

- 本文介绍apache dubbo插件的具体实现，在看代码之前，介绍一下dubbo插件干的活，以及dubbo插件的关键数据结构，再具体看看doExecute方法实现

- dubbo插件，主要作用是用来配合使用dubbo的项目，在使用dubbo协议进行调用时候，网关也可以拦截请求进行转发，相当于为了dubbo项目做了适配，而在dubbo里面，有一个元数据的概念，里面保存了使用dubbo进行rpc的时候，双方的接口信息，所以soul在实现dubbo协议的时候，也对其元数据做了适配，以此保证rpc的正确性。

  ```java
  
      protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        // 获取dubbo调用的相关数据校验
          String body = exchange.getAttribute(Constants.DUBBO_PARAMS);
          SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
          assert soulContext != null;
          MetaData metaData = exchange.getAttribute(Constants.META_DATA);
          if (!checkMetaData(metaData)) {
              assert metaData != null;
              log.error(" path is :{}, meta data have error.... {}", soulContext.getPath(), metaData.toString());
              exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
              Object error = SoulResultWrap.error(SoulResultEnum.META_DATA_ERROR.getCode(), SoulResultEnum.META_DATA_ERROR.getMsg(), null);
              return WebFluxResultUtils.result(exchange, error);
          }
          if (StringUtils.isNoneBlank(metaData.getParameterTypes()) && StringUtils.isBlank(body)) {
              exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
              Object error = SoulResultWrap.error(SoulResultEnum.DUBBO_HAVE_BODY_PARAM.getCode(), SoulResultEnum.DUBBO_HAVE_BODY_PARAM.getMsg(), null);
              return WebFluxResultUtils.result(exchange, error);
          }
        // 最终通过dubbo泛化调用得到返回值
          final Mono<Object> result = dubboProxyService.genericInvoker(body, metaData, exchange);
          return result.then(chain.execute(exchange));
      }
  ```

- 可以看到，插件的主要部分是进行调用接口的相关参数做处理，核心的地方在处理完成之后使用泛化调用对真实后端服务进行调用的部分，看看soul与dubbo集成时，在泛化调用的时候都做了什么

  ```java
  public Mono<Object> genericInvoker(final String body, final MetaData metaData, final ServerWebExchange exchange) throws SoulException {
          // issue(https://github.com/dromara/soul/issues/471), add dubbo tag route
    // 确定路由标签
          String dubboTagRouteFromHttpHeaders = exchange.getRequest().getHeaders().getFirst(Constants.DUBBO_TAG_ROUTE);
          if (StringUtils.isNotBlank(dubboTagRouteFromHttpHeaders)) {
              RpcContext.getContext().setAttachment(CommonConstants.TAG_KEY, dubboTagRouteFromHttpHeaders);
          }
    // 从元信息中获取调用路径，得到ReferenceConfig并对其进行初始化判断。
          ReferenceConfig<GenericService> reference = ApplicationConfigCache.getInstance().get(metaData.getPath());
          if (Objects.isNull(reference) || StringUtils.isEmpty(reference.getInterface())) {
              ApplicationConfigCache.getInstance().invalidate(metaData.getPath());
              reference = ApplicationConfigCache.getInstance().initRef(metaData);
          }
    // 获取泛化调用对象genericService，对之前从http的body中获取的dubbo调用的相关参数进行检查，最终得到调用的键值对进行远程调用
          GenericService genericService = reference.get();
          Pair<String[], Object[]> pair;
          if (ParamCheckUtils.dubboBodyIsEmpty(body)) {
              pair = new ImmutablePair<>(new String[]{}, new Object[]{});
          } else {
              pair = dubboParamResolveService.buildParameter(body, metaData.getParameterTypes());
          }
          CompletableFuture<Object> future = genericService.$invokeAsync(metaData.getMethodName(), pair.getLeft(), pair.getRight());
          return Mono.fromFuture(future.thenApply(ret -> {
              if (Objects.isNull(ret)) {
                  ret = Constants.DUBBO_RPC_RESULT_EMPTY;
              }
              exchange.getAttributes().put(Constants.DUBBO_RPC_RESULT, ret);
              exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());
              return ret;
          })).onErrorMap(exception -> exception instanceof GenericException ? new SoulException(((GenericException) exception).getExceptionMessage()) : new SoulException(exception));
      }
  ```

- 至此网关收到客户端请求后，使用dubbo进行一次远程调用的过程就结束了，大体流程相对比较简单，在调用之前先进行参数的校验，组合处理，之后直接使用泛化调用来拿到实际结果。

