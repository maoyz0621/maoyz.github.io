# 1 熔断降级设计理念

`sentinel-core 核心模块，限流、降级、系统保护等都在这里实现`
`sentinel-dashboard 控制台模块，可以对连接上的sentinel客户端实现可视化的管理`
`sentinel-transport 传输模块，提供了基本的监控服务端和客户端的API接口，以及一些基于不同库的实现`
`sentinel-extension 扩展模块，主要对DataSource进行了部分扩展实现`
`sentinel-adapter 适配器模块，主要实现了对一些常见框架的适配`

+ 通过并发线程数进行限制
+ 通过响应时间对资源进行降级

默认异常  BlockException

# 2 JAVA使用

```
    List<FlowRule> flowRules = new ArrayList<>();
    FlowRule flowRule = new FlowRule();
    flowRule.setResource("hello");
    flowRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
    flowRule.setCount(10);
    flowRules.add(flowRule);
    FlowRuleManager.loadRules(flowRules);


    try (Entry entry = SphU.entry("hello")) {
        // todo
    } catch (BlockException e) {
        // todo
    }

```

# 3 注解@SentinelResource使用

## 依赖

```
    <dependency>
        <groupId>com.alibaba.csp</groupId>
        <artifactId>sentinel-annotation-aspectj</artifactId>
        <version>${sentinel.version}</version>
    </dependency>

    // 使用
    @Bean
    public SentinelResourceAspect sentinelResourceAspect() {
        return new SentinelResourceAspect();
    }
```

## 参数说明

+ `value`：资源名称，必需项（不能为空）
+ `entryType`：entry 类型，可选项（默认为 `EntryType.OUT`）
+ `blockHandler` / `blockHandlerClass`: blockHandler 对应处理 `BlockException` 的函数名称，可选项。`blockHandler` 函数访问范围需要是 `public`，返回类型需要与原方法相匹配，参数类型需要和原方法相匹配并且最后加一个额外的参数，类型为 `BlockException`。blockHandler 函数默认需要和原方法在同一个类中。若希望使用其他类的函数，则可以指定 `blockHandlerClass` 为对应的类的 Class 对象，注意对应的函数必需为 `static` 函数，否则无法解析。
+ `fallback`：fallback 函数名称，可选项，用于在抛出异常的时候提供 `fallback` 处理逻辑。`fallback` 函数可以针对所有类型的异常（除了 `exceptionsToIgnore` 里面排除掉的异常类型）进行处理. fallback 函数签名和位置要求：
    1. 返回值类型必须与原函数返回值类型一致；
    2. 方法参数列表需要和原函数一致，或者可以额外多一个 `Throwable` 类型的参数用于接收对应的异常。
    3. fallback 函数默认需要和原方法在同一个类中。若希望使用其他类的函数，则可以指定 fallbackClass 为对应的类的 Class 对象，注意对应的函数必需为 static 函数，否则无法解析。
+ `defaultFallback`（since 1.6.0）：默认的 `fallback` 函数名称，可选项，通常用于通用的 `fallback` 逻辑（即可以用于很多服务或方法）。默认 `fallback` 函数可以针对所有类型的异常（除了 `exceptionsToIgnore` 里面排除掉的异常类型）进行处理。若同时配置了 `fallback` 和 `defaultFallback`，则只有 fallback 会生效。defaultFallback 函数签名要求：
    1. 返回值类型必须与原函数返回值类型一致；
    2. 方法参数列表需要为空，或者可以额外多一个 Throwable 类型的参数用于接收对应的异常。
    3. `defaultFallback` 函数默认需要和原方法在同一个类中。若希望使用其他类的函数，则可以指定 fallbackClass 为对应的类的 Class 对象，注意对应的函数必需为 static 函数，否则无法解析。
+ `exceptionsToIgnore`（since 1.6.0）：用于指定哪些异常被排除掉，不会计入异常统计中，也不会进入 fallback 逻辑中，而是会原样抛出


# 4 支持规则:流量控制规则、熔断降级规则、系统保护规则、来源访问控制规则 和 热点参数规则

## 流量控制规则 (FlowRule)
```
    private void initFlowQpsRule() {
        List<FlowRule> rules = new ArrayList<>();
        FlowRule rule = new FlowRule(resourceName);
        // set limit qps to 20
        rule.setCount(20);
        rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        rule.setLimitApp("default");
        rules.add(rule);
        FlowRuleManager.loadRules(rules);
    }
```

## 熔断降级规则 (DegradeRule)
```
    private void initDegradeRule() {
        List<DegradeRule> rules = new ArrayList<>();
        DegradeRule rule = new DegradeRule();
        rule.setResource(KEY);
        // set threshold RT, 10 ms
        rule.setCount(10);
        rule.setGrade(RuleConstant.DEGRADE_GRADE_RT);
        rule.setTimeWindow(10);
        rules.add(rule);
        DegradeRuleManager.loadRules(rules);
    }
```

## 系统保护规则 (SystemRule)
```
    private void initSystemRule() {
        List<SystemRule> rules = new ArrayList<>();
        SystemRule rule = new SystemRule();
        rule.setHighestSystemLoad(10);
        rules.add(rule);
        SystemRuleManager.loadRules(rules);
    }
```

## 访问控制规则 (AuthorityRule)
```
    AuthorityRule rule = new AuthorityRule();
    rule.setResource("test");
    rule.setStrategy(RuleConstant.AUTHORITY_WHITE);
    rule.setLimitApp("appA,appB");
    AuthorityRuleManager.loadRules(Collections.singletonList(rule));
```

## 热点规则 (ParamFlowRule)
```
    <dependency>
        <groupId>com.alibaba.csp</groupId>
        <artifactId>sentinel-parameter-flow-control</artifactId>
        <version>1.6.1</version>
    </dependency>

    ParamFlowRule rule = new ParamFlowRule(resourceName)
                        .setParamIdx(0)
                        .setCount(5);
    // 针对 int 类型的参数 PARAM_B，单独设置限流 QPS 阈值为 10，而不是全局的阈值 5.
    ParamFlowItem item = new ParamFlowItem()
                        .setObject(String.valueOf(PARAM_B))
                        .setClassType(int.class.getName())
                        .setCount(10);
    rule.setParamFlowItemList(Collections.singletonList(item));

    ParamFlowRuleManager.loadRules(Collections.singletonList(rule));
```

# 5 规则持久化

存在内存中的。即如果应用重启，这个规则就会失效。通过实现 DataSource 接口的方式，来自定义规则的存储数据源。
1. 整合动态配置系统，如 ZooKeeper、Nacos 等，动态地实时刷新配置规则
2. 结合 RDBMS、NoSQL、VCS 等来实现该规则
3. 配合 Sentinel Dashboard 使用