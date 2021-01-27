---
title: soul源码阅读-sofa插件使用
date: 2021-01-27 13:05:22
tags:
- [网关]
- [soul]
- [Java]
categories: 
- [网关, soul]
- [Java]
---

## 启动

- 参考官网介绍，在soul-bootstarp中，打开sofa相关依赖，启动soul-admin和soul-bootstrap，在启动example下soul-examples-sofa
- 在启动过程中遇到sofa注解的接口失败的问题，debug源码的时候发现，接口在注册的时候，整体流程如下
<!--more-->
![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1610993185983-0f4bcb04-0e41-4567-8be8-1ef198711f82-20210127125800421.png)

- SofaServiceBeanPostProcessor.handler
- RegisterUtils.doRegister
- OkHttpTools.getInstance.post--http#post方法调用soul-admin进行注册
- SoulClientController.registerSofaRpc
- SoulClientRegisterService.registerSofa
- 在启动的时候，sofa插件利用Spring的bean后置处理器机制，在创建一个bean的时候，通过http调用soul-admin将插件注册到网关中，然而后面报错的具体点却是，在对插件元数据进行入库持久化的时候，报错了，具体报错的点在这

![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1610993098310-28185fc1-df7c-4624-bba9-4c9b3b95d8c9-20210127125806971.png)

- 这个方法使用serviceName和method进行查询，在sofa的例子中，使用的是dubboTest类型，而如果你之前用了dubbo例子，会先将dubbo例子的接口注册的时候持久化其元数据，sofa例子的参数与dubbo的参数是一样的，而这里进行注册的时候，上述红框代码使用serviceName和method查询的时候返回2条数据，导致最终注册失败。这里查出来2条数据，具体操作是在dubbo例子中先启动一次，然后修改配置文件的appName属性在启动一次，就能注册2份appName不一样其他都一样的数据。
- 解决了问题之后，我们再进行调用测试
  ![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1610993139171-1621e479-aeb8-4d7d-9a03-0c380e784c38-20210127125814774.png)