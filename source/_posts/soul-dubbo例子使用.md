---
title: soul-dubbo例子使用
date: 2021-01-27 13:05:37
tags: 
- [网关]
- [soul]
- [Java]
categories: 
- [网关, soul]
- [Java]
---

### 运行项目

依次启动soul-admin，soul-bootstrap，soul-examples-apache-dubbo-service三个服务，观察apache-dubbo的日志，启动的时候就会将配置的接口注册到网关中，项目中使用**@SoulDubboClient**注解定义了网关要代理的路径，注解本身，是利用Spring的beanPostProcesstor特性进行加工处理，当然也可以通过控制台在页面进行操作。

<!--more-->

### 访问网关

- 因为网关作为所有流量的入口，我们此时配置好了网关管理的API之后，访问网关看看效果

```
localhost:9195/dubbo/findAll
```

- 此时正确得到了结果，查看soul-bootstrap的日志，显示成功匹配了dubbo插件，网关成功对我们请求做了代理。

![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1610810364234-6af00b34-301f-4bcd-a585-b08a52373b01-20210127130101002.png)

- 再来试试他的hystrix限流插件，将Netflix的hystrix以插件形式做了集成，首先打开插件，然后进行如下配置。

![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1610810687415-ec7fc51f-391a-4a87-a589-2a82f59fe7f2-20210127130111776.png)

![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1610810831102-5c9247f2-0be8-4dd1-8d16-1eab56b8d56d-20210127130116360.png)![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1610810842067-4b88f22f-b94d-4939-aea7-01980f489456-20210127130126483.png)

- 先将请求数限制为1，然后发起2个请求触发熔断后，去看日志提示已经打开了熔断器，并且此时在访问的时候，接口已经访问不到了。

![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1610811693960-5b3289ef-8ca4-4e57-8008-a9e02728e197-20210127130202014.png)

![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1610811792007-3527f86d-2b21-4942-8415-2fefaea79ddd-20210127130155725.png)

**关于熔断器，也有一个疑问，如果****使用semaphore作为熔断器的时候，配置了fallbackURL，是不是意味着触发熔断时会去调用降级URL，同时这个降级的URL也会去拿semaphore，如果拿不到的话回调失败？**