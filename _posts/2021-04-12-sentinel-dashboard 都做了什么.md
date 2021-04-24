```
---
layout:     post
title:      sentinel-dashboard 都做了什么
subtitle:   
date:       2021-04-12
author:     JerAxxxxxxx
header-img: img/clock_1.jpg
catalog: true
tags:
- sentinel
---
```

近日，公司打算采用阿里的 sentinel 限流组件，对整个平台的各个服务进行限流、降级处理。因为需求需要改造 sentinel-dashboard ，所以趁这个机会好好学习一下 sentinel。

### dashboard 与客户端规则的下发与读取

sentinel-dashboard 本身是一个 SpringBoot 工程 

在 dashboard 界面中，我们更改、编辑、删除流控规则时，调用的都是 `FlowControllerV1` 中的接口。

`FlowControllerV1`  类中有四个接口，分别为

- /v1/flow/rules
  从客户端获取规则。
- /v1/flow/rule
  从 dashboard 添加规则并推送至客户端。
- /v1/flow/save.json
  编辑规则
- /v1/flow/delete.json
  删除规则
#### 从 dashboard 新增规则（v1/flow/rule）
新增规则时，会调用`FlowControllerV1` 的 `/v1/flow/rule` 接口

```java
    public Result<FlowRuleEntity> apiAddFlowRule(@RequestBody FlowRuleEntity entity) {
        // 对 entity 参数做校验，如果参数都符合条件，则返回 null
        Result<FlowRuleEntity> checkResult = checkEntityInternal(entity);
        if (checkResult != null) {
            return checkResult;
        }
        entity.setId(null);
        Date date = new Date();
        entity.setGmtCreate(date);
        entity.setGmtModified(date);
        entity.setLimitApp(entity.getLimitApp().trim());
        entity.setResource(entity.getResource().trim());
        try {
            // 该方法会将 entity 对象，即限流规则存入 dashboard 的内存中
            entity = repository.save(entity);//1
			// 将 dashboard 配置的规则推送到 sentinel 客户端
            publishRules(entity.getApp(), entity.getIp(), entity.getPort()).get(5000, TimeUnit.MILLISECONDS);//2
            return Result.ofSuccess(entity);
        } catch (Throwable t) {
            Throwable e = t instanceof ExecutionException ? t.getCause() : t;
            logger.error("Failed to add new flow rule, app={}, ip={}", entity.getApp(), entity.getIp(), e);
            return Result.ofFail(-1, e.getMessage());
        }
    }
```

在 1 处的方法`repository.save(entity)` 中，主要是将流控规则对象存入 dashboard 的内存之中，具体的方法如下：

```java
public T save(T entity) {
        if (entity.getId() == null) {
            // 调用了 AtomicLong 的 incrementAndGet 方法
            entity.setId(nextId());
        }
    	// preProcess 方法主要是判断是否为集群模式
        T processedEntity = preProcess(entity);
        if (processedEntity != null) {
            // allRules，machineRules， appRules 为三个存储规则的map
            allRules.put(processedEntity.getId(), processedEntity);
            machineRules.computeIfAbsent(MachineInfo.of(processedEntity.getApp(), processedEntity.getIp(),
                processedEntity.getPort()), e -> new ConcurrentHashMap<>(32))
                .put(processedEntity.getId(), processedEntity);
            appRules.computeIfAbsent(processedEntity.getApp(), v -> new ConcurrentHashMap<>(32))
                .put(processedEntity.getId(), processedEntity);
        }

        return processedEntity;
    }
```

在 2 处的方法 `publishRules()` ，该方法主要是将配置的流控规则，通过 http 的方式推送至 sentinel 客户端的内存之中。 

#### 从 sentinel 客户端获取规则（/v1/flow/rules）

该接口在增删改流控规则时，都会被调用，基本保证了 sentinel 客户端与 dashboard 的规则一致。该方法比较简单，去除对参数的判断，主要为下列两个方法：

```java
		try {
            // 从 sentinel 客户端获取规则，这里也是使用的 http 的方式。
            List<FlowRuleEntity> rules = sentinelApiClient.fetchFlowRuleOfMachine(app, ip, port);
            // 更新 dashboard 内存中的规则，保证 dashboard 和客户端获取的规则一致
            rules = repository.saveAll(rules);
            return Result.ofSuccess(rules);
        } catch (Throwable throwable) {
            logger.error("Error when querying flow rules", throwable);
            return Result.ofThrowable(-1, throwable);
        }
```

这里的两个方法都比较好理解，首先通过 http 的方式，从 sentinel 客户端获取规则。然后将所有规则存入 dashboard 的内存中。一下为 saveAll() 方法，这里在添加规则调用的 `save()` 就是新增时调用的那个。

```java
public List<T> saveAll(List<T> rules) {
        // TODO: check here.
    	// 清空 map
        allRules.clear();
        machineRules.clear();
        appRules.clear();

        if (rules == null) {
            return null;
        }
        List<T> savedRules = new ArrayList<>(rules.size());
        for (T rule : rules) {
            savedRules.add(save(rule));
        }
        return savedRules;
    }
```

#### 编辑规则（/v1/flow/save.json）

编辑接口的控制层代码非常多，个人感觉有些冗余，这里就不贴代码了。主要是对流控的各个参数进行校验。最后调用了 `publishRules()` 方法，同样是确保 dashboard 和 sentinel 客户端的规则是一致的。

#### 删除规则（/v1/flow/delete.json）

删除接口基本上和编辑规则的接口没什么区别，大量的参数校验，以及调用`publishRules()` 方法，确保规则一致。



### 总结

至此，dashboard 的代码已经看完，虽然 sentinel-dashboard 并不是一个生产环境开箱即用的组件，但是对于学习 sentinel 服务还是有很大的帮助的。本篇主要从控制台的各个操作，学习了 dashboard 的大体功能，下一篇将分析 **dashboard 是如何和 sentinel 建立通信**，并获取规则的。