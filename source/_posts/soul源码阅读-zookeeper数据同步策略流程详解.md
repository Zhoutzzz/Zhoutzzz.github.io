---
title: soul源码阅读-zookeeper数据同步策略流程详解
date: 2021-01-27 13:04:05
tags:
- [网关]
- [soul]
- [Java]
categories: 
- [网关, soul]
- [Java]
---

## 前言

- 这次讲解soul中使用zookeeper网关同步数据的流程，照旧参考官方文档的说明。

- > 依赖zookeeper的 watch 机制，soul-web会监听配置的节点，soul-admin在启动的时候，会将数据全量写入 zookeeper，后续数据发生变更时，会增量更新zookeeper的节点，与此同时，soul-web 会监听配置信息的节点，一旦有信息变更时，会更新本地缓存
<!--more-->
- 我们画一下这个流程图：

- ```mermaid
  graph LR
  
  a[soul-admin发起一个修改] --> b[zookeeper]
  b[zookeeper watch, 修改数据发送到web] --> c[soul-web接收进行处理] 
  ```

  ## admin像zk发送数据

- 依旧是类似的实现，使用zk当然就使用对应的ZookeeperDataChangedListener来作为zk的客户端发送最新数据了，挑2个方法来简单介绍就好：

  ```java
  public void onMetaDataChanged(final List<MetaData> changed, final DataEventTypeEnum eventType) {
          for (MetaData data : changed) {
              String metaDataPath = ZkPathConstants.buildMetaDataPath(URLEncoder.encode(data.getPath(), "UTF-8"));
              // delete
              if (eventType == DataEventTypeEnum.DELETE) {
                  deleteZkPath(metaDataPath);
                  continue;
              }
              // create or update
              insertZkNode(metaDataPath, data);
          }
      }
  
      @Override
      public void onPluginChanged(final List<PluginData> changed, final DataEventTypeEnum eventType) {
          for (PluginData data : changed) {
              String pluginPath = ZkPathConstants.buildPluginPath(data.getName());
              // delete
              if (eventType == DataEventTypeEnum.DELETE) {
                  deleteZkPathRecursive(pluginPath);
                  String selectorParentPath = ZkPathConstants.buildSelectorParentPath(data.getName());
                  deleteZkPathRecursive(selectorParentPath);
                  String ruleParentPath = ZkPathConstants.buildRuleParentPath(data.getName());
                  deleteZkPathRecursive(ruleParentPath);
                  continue;
              }
              //create or update
              insertZkNode(pluginPath, data);
          }
      }
  ```

  

- 也很明显，对于多级节点的情况，递归删除树下的子节点然后将新数据添加就好，单节点就直接删除，**其实zk的的数据同步相比nacos，http长轮询的方式确实要简单的多，可能也是因为zk本身的强一致性简化了这里的实现**。

## web更新缓存

- 那么，在zk接收了数据之后，zk就会往soul发送数据了，之前也说了，soul中的这几种数据同步方式结构上很相似，所以我们可以直接看收到zk数据之后具体的处理操作就好，实际处理是在`ZookeeperSyncDataService`	类中，soul中代码，做了高度封装，这里以几个数据类型的处理方法作为示例，其他数据类型处理方式基本相同

  ```java
  // 初始化时先从zk获取元数据进行更新，更新完毕在创建监听器监听后续修改
  private void watchMetaData() {
          final String metaDataPath = ZkPathConstants.META_DATA;
    // 先获取子节点
          List<String> childrenList = zkClientGetChildren(metaDataPath);
          if (CollectionUtils.isNotEmpty(childrenList)) {
              childrenList.forEach(children -> {
                // 缓存对应的数据
                  String realPath = buildRealPath(metaDataPath, children);
                  cacheMetaData(zkClient.readData(realPath));
                  subscribeMetaDataChanges(realPath);
              });
          }
          subscribeChildChanges(ConfigGroupEnum.META_DATA, metaDataPath, childrenList);
      }
  
  // 获取zk中对应数据的子节点
  private List<String> zkClientGetChildren(final String parent) {
          if (!zkClient.exists(parent)) {
              zkClient.createPersistent(parent, true);
          }
          return zkClient.getChildren(parent);
      }
  
  // 构建缓存的路径
      private String buildRealPath(final String parent, final String children) {
          return parent + "/" + children;
      }
  
  // 缓存元数据
      private void cacheMetaData(final MetaData metaData) {
          Optional.ofNullable(metaData).ifPresent(data -> metaDataSubscribers.forEach(e -> e.onSubscribe(metaData)));
      }
  
  // 订阅元数据修改的监听器
      private void subscribeMetaDataChanges(final String realPath) {
          zkClient.subscribeDataChanges(realPath, new IZkDataListener() {
              @Override
              public void handleDataChange(final String dataPath, final Object data) {
                  cacheMetaData((MetaData) data);
              }
  
              @SneakyThrows
              @Override
              public void handleDataDeleted(final String dataPath) {
                  final String realPath = dataPath.substring(ZkPathConstants.META_DATA.length() + 1);
                  MetaData metaData = new MetaData();
                  metaData.setPath(URLDecoder.decode(realPath, StandardCharsets.UTF_8.name()));
                  unCacheMetaData(metaData);
              }
          });
      }
  
  // 相关数据类型下的子节点监听器，如果任意数据下的子节点发生变动就会收到数据同步缓存
      private void subscribeChildChanges(final ConfigGroupEnum groupKey, final String groupParentPath, final List<String> childrenList) {
          switch (groupKey) {
              ...
              case META_DATA:
                  zkClient.subscribeChildChanges(groupParentPath, (parentPath, currentChildren) -> {
                      if (CollectionUtils.isNotEmpty(currentChildren)) {
                          final List<String> addSubscribePath = addSubscribePath(childrenList, currentChildren);
                          addSubscribePath.stream().map(children -> {
                              final String realPath = buildRealPath(parentPath, children);
                              cacheMetaData(zkClient.readData(realPath));
                              return realPath;
                          }).forEach(this::subscribeMetaDataChanges);
                      }
                  });
                  break;
              default:
                  throw new IllegalStateException("Unexpected groupKey: " + groupKey);
          }
      }
  ```

  

## 结尾

整个zk同步策略的流程就结束了，到此soul中所有的数据同步方式的流程就都介绍了，整体来说，数据同步部分的设计确实很好，实现上做了高度统一，在数据同步的时候使用发布订阅模式，不同类型处理方式都基本相同，无论是代码阅读还是使用都很方便，而且内部的实现细节上面做了高度封装，基本没有什么冗余代码，只能说两个字，优雅。