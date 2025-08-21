---
description: 项目分库改造全过程记录，详细介绍基于ShardingSphere-JDBC的分库实现方案
title: 基于ShardingSphere-JDBC的分库实现
tag:
  - 工作
  - MySQL
sidebar: true
comment: true
recommend: 1
---
# 🗂 基于 ShardingSphere-JDBC 的分库实现

> 📒 本文记录了项目分库改造的全过程，涵盖 **背景、选型、实现方案、核心代码** 等关键环节。

------

## 1️⃣ 背景与决策：为何选择分库？

### 📌 项目现状与挑战

当前系统服务 **7 个租户**，面临以下问题：

1. 🏷 **数据隔离不足**：部分核心表缺少租户ID字段，导致数据混合存储。
2. 📊 **数据规模庞大**：迁移完成后，保单数据量达 **百万级**，账单数据量达 **亿级**。
3. ⚠️ **扩展性瓶颈**：在单库架构下，数据库将承受巨大压力。

### ✅ 最终决策

基于 **缺乏租户字段** + **数据规模快速增长** 的双重压力，团队选择了 **物理分库方案**，实现 **租户级数据隔离** 并提升系统可扩展性。

------

## 2️⃣ 组件选型：为何是 ShardingSphere？

在分库组件的选型中，我们评估了三条路径：

- 🔌 **应用层集成**（ShardingSphere）
- 🔗 **代理层中间件**（Mycat）
- ☁️ **云数据库解决方案**

最终选择 **ShardingSphere 5.5.0**，原因包括：

- ⚙️ **运维复杂度低**：应用内集成，符合现有运维体系。
- 🛡 **系统稳定性高**：无额外代理层，减少潜在故障点。
- 🧩 **代码侵入性低**：作为 JDBC 驱动接入，对业务改造成本最小。

------

## 3️⃣ 具体实现方案

### 🔧 技术环境

- **JDK**：21
- **Spring Boot**：3.0
- **ShardingSphere**：5.5.0

### 📐 分库策略

- **HintManager 强制路由**
   由于部分表无租户字段（`itnt_code`），无法按标准分片键分库，采用 **HintManager** 显式指定数据源。

### 🔄 租户标识传递链路

1. 🌐 **前端接入层**：HTTP 请求强制携带 `Tenant Code`。
2. 🛡 **过滤器层**：解析 `Tenant Code` 并写入 `ThreadLocal`。
3. 🔀 **异步任务**：改造 `ThreadPoolTaskExecutor` 的 `TaskDecorator`，支持跨线程传递。
4. ⏱ **批处理 & 定时任务**：任务启动时显式注入租户上下文，确保访问正确数据源。

### 🧩 核心组件适配

- **Liquibase**：绕过 ShardingSphere，直接使用物理数据源，手动注册 `LiquibaseBean`。
- **DataPatch**：按租户分批执行，依赖 `HintManager` 动态切换。
- **分页查询**：手动为 `SqlSessionFactoryBean` 配置分页插件。

------

## 4️⃣ 核心代码示例

### 🏷 租户上下文管理

```
class TenantContext {
    private static final ThreadLocal<String> TENANT_CODE = new ThreadLocal<>();

    static void setTenantCode(String tenantCode) {
        TENANT_CODE.set(tenantCode);
    }
    static String getTenantCode() {
        return TENANT_CODE.get();
    }
    static void clear() {
        TENANT_CODE.remove();
    }
}
```

### 🛡 过滤器改造

```
@Slf4j
class UrlAccessTokenFilter extends GenericFilterBean {
    private static final String TENANT_CODE = "X-Tenant-Code";
    @Override
    void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpReq = (HttpServletRequest) request;
        String tenantCode = httpReq.getHeader(TENANT_CODE)?.toUpperCase();
        if (MyStringUtils.isBlank(TenantContext.tenantCode)) {
            TenantContextUtils.run(() -> chain.doFilter(new CustomizedHttpRequest(httpReq), response))
                .on(this).withTenantCode(tenantCode).go();
        } else {
            chain.doFilter(new CustomizedHttpRequest(httpReq), response);
        }
    }
}
```

### ⚙️ ShardingSphere 配置

```
@Configuration
class ShardingConfig {
    @Bean
    @Primary
    DataSource dataSource() throws SQLException {
        return ShardingSphereDataSourceFactory.createDataSource(
            "sharding_db", createModeConfiguration(),
            getPhysicalDataSources(datasourcesConf),
            Collections.singleton(createShardingRuleConfig()),
            getShardingSphereProperties());
    }
    private static ShardingRuleConfiguration createShardingRuleConfig() {
        ShardingRuleConfiguration config = new ShardingRuleConfiguration();
        createTableRules(config);
        config.setDefaultDatabaseShardingStrategy(new HintShardingStrategyConfiguration("thread_local_hint"));
        Properties hintAlgorithmProps = new Properties();
        hintAlgorithmProps.setProperty("algorithm-expression", "ds_${value}");
        hintAlgorithmProps.setProperty("check-table-metadata-enabled", "false");
        config.getShardingAlgorithms().put("thread_local_hint", new AlgorithmConfiguration("HINT_INLINE", hintAlgorithmProps));
        return config;
    }
}
```

### 🎯 AOP 注入 HintManager

```
@Aspect
@Component
@Slf4j
class ShardingAspect {
    @Pointcut("execution(* com.bytesforce.insmate..*.*Mapper.*(..)) || bean(shardingSqlSessionTemplate) || bean(shardingJdbcTemplate)")
    void daoMethods() {}
    @Around("daoMethods()")
    Object handleSharding(ProceedingJoinPoint joinPoint) throws Throwable {
        String tenantCode = TenantContext.getTenantCode();
        if (MyStringUtils.isBlank(tenantCode)) {
            throw new IllegalArgumentException("Tenant code is empty, can't find ds.");
        }
        HintManager manager = null;
        try {
            if (!HintManager.isInstantiated()) {
                manager = HintManager.getInstance();
                Integer dsSeq = TenantEnum.getByCode(tenantCode).dsSeq;
                manager.setDatabaseShardingValue(dsSeq);
            }
            return joinPoint.proceed();
        } finally {
            if (Objects.nonNull(manager)) {
                manager.close();
            }
        }
    }
}
```