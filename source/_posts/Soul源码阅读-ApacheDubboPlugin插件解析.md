---
title: Soul源码阅读-ApacheDubboPlugin插件解析
date: 2021-01-27 13:03:54
tags:
- [网关]
- [soul]
- [Java]
categories: 
- [网关, soul]
- [Java]
---

- 本文介绍apache dubbo插件的具体实现，在看代码之前，介绍一下dubbo插件干的活，以及dubbo插件的关键数据结构，再具体看看doExecute方法实现
<!--more-->
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

  