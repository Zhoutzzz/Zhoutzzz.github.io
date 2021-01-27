---
title: soul源码阅读-使用@SoulSpringMvcClient将接口注册到网关流程解析
date: 2021-01-27 13:04:36
tags:
- [网关]
- [soul]
- [Java]
categories: 
- [网关, soul]
- [Java]
---

- 在使用soul将我们编写的controller接口注册到网关，由网关统一代理时，一般情况，http方式，我们只需要使用@SoulSpringMvcClient注解标注在对应的接口上就行了，那么，我们使用了注解之后，soul是如何将我们的接口注册到网关的呢
- 先看看注解在哪些地方被使用了
<!--more-->
![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1611243592672-921784be-84e7-440b-adfb-2c5c014c4e07-20210127130021661.png)

- 我们进到这个类里面,使用了spring的bean实例化里面的后置处理器机制进行了具体处理。

```
@Slf4j
public class SpringMvcClientBeanPostProcessor implements BeanPostProcessor {

    private final ThreadPoolExecutor executorService;

    private final String url;

    private final SoulSpringMvcConfig soulSpringMvcConfig;

    /**
     * Instantiates a new Soul client bean post processor.
     *
     * @param soulSpringMvcConfig the soul spring mvc config
     */
    // 构造器里面初始化了调用url。
    public SpringMvcClientBeanPostProcessor(final SoulSpringMvcConfig soulSpringMvcConfig) {
        ValidateUtils.validate(soulSpringMvcConfig);
        this.soulSpringMvcConfig = soulSpringMvcConfig;
        // 这里到adminUrl就是admin控制台地址，后面的path就是具体到路径了
        url = soulSpringMvcConfig.getAdminUrl() + "/soul-client/springmvc-register";
        executorService = new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<>());
    }

    @Override
    public Object postProcessAfterInitialization(@NonNull final Object bean, @NonNull final String beanName) throws BeansException {
        if (soulSpringMvcConfig.isFull()) {
            return bean;
        }
        // 获取到一些常用注解
        Controller controller = AnnotationUtils.findAnnotation(bean.getClass(), Controller.class);
        RestController restController = AnnotationUtils.findAnnotation(bean.getClass(), RestController.class);
        RequestMapping requestMapping = AnnotationUtils.findAnnotation(bean.getClass(), RequestMapping.class);
        if (controller != null || restController != null || requestMapping != null) {
            根据上面获取到注解判断，获取SoulSpringMvcClient注解，之后创建新线程向soul注册接口
            SoulSpringMvcClient clazzAnnotation = AnnotationUtils.findAnnotation(bean.getClass(), SoulSpringMvcClient.class);
            String prePath = "";
            if (Objects.nonNull(clazzAnnotation)) {
                if (clazzAnnotation.path().indexOf("*") > 1) {
                    String finalPrePath = prePath;
                    executorService.execute(() -> RegisterUtils.doRegister(buildJsonParams(clazzAnnotation, finalPrePath), url,
                            RpcTypeEnum.HTTP));
                    return bean;
                }
                prePath = clazzAnnotation.path();
            }
            final Method[] methods = ReflectionUtils.getUniqueDeclaredMethods(bean.getClass());
            for (Method method : methods) {
                SoulSpringMvcClient soulSpringMvcClient = AnnotationUtils.findAnnotation(method, SoulSpringMvcClient.class);
                if (Objects.nonNull(soulSpringMvcClient)) {
                    String finalPrePath = prePath;
                    executorService.execute(() -> RegisterUtils.doRegister(buildJsonParams(soulSpringMvcClient, finalPrePath), url,
                            RpcTypeEnum.HTTP));
                }
            }
        }
        return bean;
    }

    // 通过注解里的属性构造出请求地址
    private String buildJsonParams(final SoulSpringMvcClient soulSpringMvcClient, final String prePath) {
        String contextPath = soulSpringMvcConfig.getContextPath();
        String appName = soulSpringMvcConfig.getAppName();
        Integer port = soulSpringMvcConfig.getPort();
        String path = contextPath + prePath + soulSpringMvcClient.path();
        String desc = soulSpringMvcClient.desc();
        String configHost = soulSpringMvcConfig.getHost();
        String host = StringUtils.isBlank(configHost) ? IpUtils.getHost() : configHost;
        String configRuleName = soulSpringMvcClient.ruleName();
        String ruleName = StringUtils.isBlank(configRuleName) ? path : configRuleName;
        SpringMvcRegisterDTO registerDTO = SpringMvcRegisterDTO.builder()
                .context(contextPath)
                .host(host)
                .port(port)
                .appName(appName)
                .path(path)
                .pathDesc(desc)
                .rpcType(soulSpringMvcClient.rpcType())
                .enabled(soulSpringMvcClient.enabled())
                .ruleName(ruleName)
                .registerMetaData(soulSpringMvcClient.registerMetaData())
                .build();
        return OkHttpTools.getInstance().getGson().toJson(registerDTO);
    }
}
```

- 看看具体RegisterUtils.doRegister()方法

```
public static void doRegister(final String json, final String url, final RpcTypeEnum rpcTypeEnum) {
        try {
            // 发起http请求，注意这个url，是向admin控制台发起调用
            String result = OkHttpTools.getInstance().post(url, json);
            if (AdminConstants.SUCCESS.equals(result)) {
                log.info("{} client register success: {} ", rpcTypeEnum.getName(), json);
            } else {
                log.error("{} client register error: {} ", rpcTypeEnum.getName(), json);
            }
        } catch (IOException e) {
            log.error("cannot register soul admin param, url: {}, request body: {}", url, json, e);
        }
    }
```

- doRegister唯一作用就是发起一个向soul-admin对应接口的http调用，那我们看看soul-admin

![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1611244925460-dcc8d25f-3677-40da-b993-703991845b08-20210127130028645.png)

- 也是一个很常规的controller接口，成功调用到了

![image.png](https://cdn.jsdelivr.net/gh/Zhoutzzz/picgoture/1611245076281-bb2e000a-3176-4afb-b2b8-cf368a5f8a3f-20210127130041273.png)

- 里面有两个方法**handlerSpringMvcSelector**和**handlerSpringMvcRule**， 我们看看这两个方法具体干了什么
- selector：

```
private String handlerSpringMvcSelector(final SpringMvcRegisterDTO dto) {
    // 进行一些参数的校验转换的处理
        String contextPath = dto.getContext();
        SelectorDO selectorDO = selectorService.findByName(contextPath);
        String selectorId;
        String uri = String.join(":", dto.getHost(), String.valueOf(dto.getPort()));
        if (Objects.isNull(selectorDO)) {
            selectorId = registerSelector(contextPath, dto.getRpcType(), dto.getAppName(), uri);
        } else {
            selectorId = selectorDO.getId();
            //update upstream
            String handle = selectorDO.getHandle();
            String handleAdd;
            DivideUpstream addDivideUpstream = buildDivideUpstream(uri);
            SelectorData selectorData = selectorService.buildByName(contextPath);
            if (StringUtils.isBlank(handle)) {
                handleAdd = GsonUtils.getInstance().toJson(Collections.singletonList(addDivideUpstream));
            } else {
                List<DivideUpstream> exist = GsonUtils.getInstance().fromList(handle, DivideUpstream.class);
                for (DivideUpstream upstream : exist) {
                    if (upstream.getUpstreamUrl().equals(addDivideUpstream.getUpstreamUrl())) {
                        return selectorId;
                    }
                }
                exist.add(addDivideUpstream);
                handleAdd = GsonUtils.getInstance().toJson(exist);
            }
            selectorDO.setHandle(handleAdd);
            selectorData.setHandle(handleAdd);
            // update db 保存配置到数据库
            selectorMapper.updateSelective(selectorDO);
            // submit upstreamCheck 将配置更新到本地缓存
            upstreamCheckService.submit(contextPath, addDivideUpstream);
            // publish change event.  使用spring 事件发布，发布配置更改事件到soul网关核心。
            eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.SELECTOR, DataEventTypeEnum.UPDATE,
                    Collections.singletonList(selectorData)));
        }
        return selectorId;
    }
```

- rule: 

```
public String register(final RuleDTO ruleDTO) {
    // 规则入库保存
        RuleDO ruleDO = RuleDO.buildRuleDO(ruleDTO);
        List<RuleConditionDTO> ruleConditions = ruleDTO.getRuleConditions();
        if (StringUtils.isEmpty(ruleDTO.getId())) {
            ruleMapper.insertSelective(ruleDO);
            ruleConditions.forEach(ruleConditionDTO -> {
                ruleConditionDTO.setRuleId(ruleDO.getId());
                ruleConditionMapper.insertSelective(RuleConditionDO.buildRuleConditionDO(ruleConditionDTO));
            });
        }
    // 发布修改事件
        publishEvent(ruleDO, ruleConditions);
        return ruleDO.getId();
    }

private void publishEvent(final RuleDO ruleDO, final List<RuleConditionDTO> ruleConditions) {
        SelectorDO selectorDO = selectorMapper.selectById(ruleDO.getSelectorId());
        PluginDO pluginDO = pluginMapper.selectById(selectorDO.getPluginId());

        List<ConditionData> conditionDataList =
                ruleConditions.stream().map(ConditionTransfer.INSTANCE::mapToRuleDTO).collect(Collectors.toList());
        // publish change event.
        eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.RULE, DataEventTypeEnum.UPDATE,
                Collections.singletonList(RuleDO.transFrom(ruleDO, pluginDO.getName(), conditionDataList))));
    }
```

- 只是简单的入库保存配置，然后通过监听，发布更改事件，那就要看看发布的监听具体内容了。

```
@Component
// 指定了泛型DataChangedEvent，所以结合上面的事件发布传入的参数，可以定位到soul定义的事件分发器里面
public class DataChangedEventDispatcher implements ApplicationListener<DataChangedEvent>, InitializingBean {

    private ApplicationContext applicationContext;

    //内部注入了spring 监听来进行事件发送
    private List<DataChangedListener> listeners;

    public DataChangedEventDispatcher(final ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    @Override
    @SuppressWarnings("unchecked")
    public void onApplicationEvent(final DataChangedEvent event) {
        for (DataChangedListener listener : listeners) {
            switch (event.getGroupKey()) {
                case APP_AUTH:
                    listener.onAppAuthChanged((List<AppAuthData>) event.getSource(), event.getEventType());
                    break;
                case PLUGIN:
                    listener.onPluginChanged((List<PluginData>) event.getSource(), event.getEventType());
                    break;
                case RULE:
                    listener.onRuleChanged((List<RuleData>) event.getSource(), event.getEventType());
                    break;
                case SELECTOR:
                    listener.onSelectorChanged((List<SelectorData>) event.getSource(), event.getEventType());
                    break;
                case META_DATA:
                    listener.onMetaDataChanged((List<MetaData>) event.getSource(), event.getEventType());
                    break;
                default:
                    throw new IllegalStateException("Unexpected value: " + event.getGroupKey());
            }
        }
    }

    @Override
    public void afterPropertiesSet() {
        Collection<DataChangedListener> listenerBeans = applicationContext.getBeansOfType(DataChangedListener.class).values();
        this.listeners = Collections.unmodifiableList(new ArrayList<>(listenerBeans));
    }

}
```

- 匹配到对应的类型时，走了具体的监听事件，以selector的为例子，listener.onSelectorChanged，因为soul的数据同步，默认使用的时http长轮询来进行的，最终定位到了AbstractDataChangedListener.onSelectorChanged方法

```
    @Override
    public void onSelectorChanged(final List<SelectorData> changed, final DataEventTypeEnum eventType) {
        if (CollectionUtils.isEmpty(changed)) {
            return;
        }
        // 更新缓存
        this.updateSelectorCache();
        this.afterSelectorChanged(changed, eventType);
    }

    protected void afterSelectorChanged(final List<SelectorData> changed, final DataEventTypeEnum eventType) {
    }   
```

- 这个afterSelectorChanged的方法默认是个空的实现，往下看具体实现，发现只有一个，就是HttpLongPollingDataChangedListener，实际执行长轮询同步数据的类（模版模式的再一次应用）。

```
protected void afterSelectorChanged(final List<SelectorData> changed, final DataEventTypeEnum eventType) {
    //调度执行任务
        scheduler.execute(new DataChangeTask(ConfigGroupEnum.SELECTOR));
    }
```

- 看看这个实现了Runnable的DataChangeTask中的run方法干了什么，结合官方文档解释，长轮询策略下，soul-web请求admin，admin会将数据同步请求放入阻塞队列中延迟处理，如果60秒没有配置更新请求，会触发一次将队列头部元素出队进行处理，然后返回响应，如果有配置修改请求发生，会将队列中的所有请求出队依次处理响应给soul-web。

```
@Override
        public void run() {
            for (Iterator<LongPollingClient> iter = clients.iterator(); iter.hasNext();) {
                LongPollingClient client = iter.next();
                iter.remove();
                client.sendResponse(Collections.singletonList(groupKey));
                log.info("send response with the changed group,ip={}, group={}, changeTime={}", client.ip, groupKey, changeTime);
            }
        }
```

- 同样的，LongPollingClient也是个实现runnable的方法

```
@Override
        public void run() {
            this.asyncTimeoutFuture = scheduler.schedule(() -> {
                // 出队响应
                clients.remove(LongPollingClient.this);
                List<ConfigGroupEnum> changedGroups = compareChangedGroup((HttpServletRequest) asyncContext.getRequest());
                sendResponse(changedGroups);
            }, timeoutTime, TimeUnit.MILLISECONDS);
            clients.add(this);
        }
```

- 因为我们在使用@SoulSpringMvcClient注册我们的接口的时候，项目启动不需要进行同步操作，所以这里复用了响应方法，响应在一开始说的doRegister方法发出的请求。至此，项目启动时利用@SoulSpringMvcClient向网关注册接口的流程就走完了。