---
title: soul源码阅读-springCloud插件使用
date: 2021-01-27 13:05:07
tags:
- [网关]
- [soul]
- [Java]
categories: 
- [网关, soul]
- [Java]
---

## 启动

- 在soul-bootstarp的pom中，打开Spring cloud的依赖。
- 参考官网介绍启动soul-admin、soul-bootstrap、eureka-server，再启动example下soul-examples-spring cloud
<!--more-->
- 日志显示spring cloud接口成功注册到了网关

![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1611074821933-1fb8b0f7-436f-4220-9621-bab5693c87d9-20210127125318648.png)

- 访问网关，成功拿到响应结果。

![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1611152838523-134c6dde-9264-4fb5-ab9b-e698e7816362-20210127125302963.png)

- 其实之前网关一直请求不到的，后来追溯源码，理清楚了插件调用过程，才发现，就是pom中spring cloud相关依赖没加。。。