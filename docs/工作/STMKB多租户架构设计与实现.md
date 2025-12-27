---
description: 当子公司业务接入，系统如何从单租户平滑演进为多租户？本文深度复盘 STMKB 项目的改造过程，从存储方案的成本与性能权衡，到 Spring Boot 动态数据源、Elasticsearch 及文件存储的全链路隔离实现，为你提供一套可落地的企业级解决方案。
title:  系统演进实战：详解多租户架构下的数据隔离与全链路设计
tag:
  - 工作
  - JAVA
sidebar: true
comment: true
recommend: 1
sticky: 3
---
# 🚀 STMKB多租户架构设计与实现

## 背景

KB客户子公司AB也想要使用我们的系统，系统数据需要隔离，且需要实现多租户数据隔离。

**提供以下方案：**

| 方案       | 解决方案                                     | 优点                                                         | 缺点                                                         |
| :--------- | :------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **方案一** | 同一磁盘，不同文件夹（保持现有存储设备资源） | 1. **节省成本**：共享同一磁盘有助于降低硬件采购和维护成本，减少存储资源浪费。 2. **配置变更简单，风险低**：仅需在同一磁盘中创建不同目录，无需对系统架构进行大调整。 3. **维护方便**：磁盘容量大（目前仅使用15%），维护便捷。 | 1. **性能瓶颈**：若多个租户有高存储访问需求，共享同一磁盘的读写资源可能导致性能瓶颈，影响整体存储速度和响应性。 2. **安全性和隔离性较低**：尽管每个租户有独立目录，但物理磁盘是共享的。若权限管理不当，可能存在租户间数据访问风险。 |
| **方案二** | 不同磁盘，不同文件夹（需额外存储设备资源）   | 1. **性能优化**：每个租户使用独立磁盘，避免共享磁盘的性能瓶颈，提升存储操作速度和响应性。 2. **更好的隔离性和扩展性**：每个租户数据存储在独立物理磁盘上，提供更强的隔离性，且每个磁盘有独立的存储空间，管理更清晰、扩展性更强。 | 1. **成本更高**：每个租户需单独磁盘，增加了硬件采购和维护成本。 2. **配置和维护复杂度增加**：配置和维护更复杂，新增租户需同步挂载磁盘配置。 |

简要说明

- **方案一** 侧重于成本节约和简易性，适用于对性能和隔离性要求不高的场景。
- **方案二** 更注重性能、安全性和扩展性，适用于对存储性能和数据隔离有较高要求的场景，但成本和管理复杂度也相应提高。

## 系统分析

   为了实现租户之间的业务数据隔离，需求涵盖以下方面：数据库（DB）、Elasticsearch（ES）和文件存储（File）。

| *模块* | *涉及内容*                                                   |
| ------ | ------------------------------------------------------------ |
| 2B     | 数据隔离: Current  tenant user仅可访问操作Current tenant数据 |
| 2C     | 数据隔离: Customer购买Online  product通过Domain区分tenant    |

## 系统设计

### 多租户访问

#### 2B租户访问流程

![image-20251105220839914](.\image\image-20251105220839914.png)

#### Online租户访问流程

![image-20251105220856962](.\image\image-20251105220856962.png)

#### Frontend初始化tenant info

![image-20251105220914126](.\image\image-20251105220914126.png)

#### 过滤器TenantCodeFilter

> 过滤器(Filter)先执行，拦截器(Interceptor)后执行。

Step 1: retrieve from request domain (User)

Step 2:  retrieve from request header (Open API)

Step 3: retrieve from request parameter (DEV)

Step 4: for dev

```groovy
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
@Slf4j
class TenantCodeFilter extends GenericFilterBean {

    @Override
    void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        try {
            String tenantCode = TenantRequestUtils.getTenantCode((HttpServletRequest) request)
            log.info("request url: ${((HttpServletRequest) request).getRequestURL()}")
            if (tenantCode != null) {
                TenantContext.setCurrentTenant(tenantCode)
            }
            chain.doFilter(request, response)
        } finally {
            // 请求处理完成后清除上下文
            TenantContext.clear()
        }
    }
}

package com.bytesforce.insmate.configs

import com.bytesforce.insmate.pub.PAConst
import com.bytesforce.insmate.tenant.config.TenantConf
import com.bytesforce.pub.Bean
import com.bytesforce.pub.EnvUtils
import com.bytesforce.pub.MyStringUtils

import javax.servlet.http.HttpServletRequest
import java.util.concurrent.ConcurrentHashMap

class TenantRequestUtils {

    private static volatile Map<String, String> TENANT_CACHE = new ConcurrentHashMap<>()

    static String getTenantCode(HttpServletRequest request) {
        String tenantCode

        // Step 1: retrieve from request domain
        String domain = getDomainName(request?.getRequestURL()?.toString())
        tenantCode = matchTenant(domain)
        if (MyStringUtils.isNotBlank(tenantCode)) {
            return tenantCode
        }

        // Step 2: retrieve from request header
        Enumeration<String> headers = request.getHeaders("x-insmate-itntcode")
        while (headers.hasMoreElements()) {
            tenantCode = headers.nextElement()
        }
        if (MyStringUtils.isNotBlank(tenantCode)) {
            return tenantCode
        }

        // Step3: retrieve from request parameter
        tenantCode = request.getParameter("itntCode")
        if (MyStringUtils.isNotBlank(tenantCode)) {
            return tenantCode
        }

        // Step4: for dev
        if (EnvUtils.dev()) {
            return PAConst.STMAB_TENANT_CODE
        }
        return null
    }

    private static String getDomainName(String url) throws URISyntaxException {
        if (!url.startsWith("http://") && !url.startsWith("https://")) {
            url = "https://" + url
        }
        URI uri = new URI(url)
        String domain = uri.getHost()
        return domain.startsWith("www.") ? domain.substring(4) : domain
    }

    private static String matchTenant(String domain) {
        return TENANT_CACHE.computeIfAbsent(
                domain,
                { d ->
                    Bean.get(TenantConf).tenants?.find { tenantCode, tenant ->
                        tenant.domains?.collect { it.trim() }?.contains(d.trim())
                    }?.key
                }
        )
    }

}
```

TenantContext

```groovy
class TenantContext {
    private static final ThreadLocal<String> currentTenant = new ThreadLocal<>()

    static String getCurrentTenant() {
        return currentTenant.get()
    }

    static void setCurrentTenant(String tenant) {
        MDC.put('tenant', tenant)
        currentTenant.set(tenant)
    }

    static void clear() {
        MDC.remove('tenant')
        currentTenant.remove()
    }
}
```

TenantContextUtils

```groovy
class TenantContextUtils<T> {
    private TenantContextUtils() {}

    private Closure closure
    private Object delegate
    private String originalItntCode
    private String newItntCode

    static <T> TenantContextUtils<T> run(Closure closure) {
        return new TenantContextUtils<>(closure: closure)
    }

    TenantContextUtils<T> on(Object delegate) {
        this.delegate = delegate
        this
    }

    TenantContextUtils<T> withItntCode(String itntCode) {
        ParamAssert.notEmpty(itntCode, "Tenant code should be provided")
        ParamAssert.isTrue(
                Bean.get(TenantConf)?.tenantCodes()?.any { it == itntCode },
                "Tenant code is invalid: ${itntCode}".toString()
        )
        newItntCode = itntCode
        this
    }

    T go() {
        if (delegate == null) {
            delegate = this
        }
        originalItntCode = TenantContext.currentTenant
        TenantContext.setCurrentTenant(newItntCode)
        try {
            return (T) MyClosureUtils.run(closure).on(delegate).go()
        } finally {
    TenantContext.setCurrentTenant(originalItntCode)
        }
    }
}
```

### Database隔离

1.Platform

![image-20251105223700924](.\image\image-20251105223700924.png)

1. 数据迁移

a) STMKB数据库保持不变: stmkb_prod_db

b) STMAB数据库新增: stmab_prod_db

2. 权限控制

a) STMKB user保持不变, 配置STMKB user仅可操作stmkb_prod_db

b) STMAB user新增: 配置STMAB user仅可操作stmab_prod_db

3. 数据隔离

- application.yml

​       在application.yml中改造数据库连接配置, 适配多租户多数据源

```
  datasource:
    tenants:
      STMKB:
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://127.0.0.1:3306/stmkbgrp_db?characterEncoding=utf8&ssl-mode=DISABLED&useLegacyDatetimeCode=false&serverTimezone=Asia/Singapore
        username: stmkb
        password: Insteko_666
        hikari:
          pool-name: stmkb-hikari-connection-pool
          leak-detection-threshold: 30000
      STMAB:
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://127.0.0.1:3306/stmabgrp_db?characterEncoding=utf8&ssl-mode=DISABLED&useLegacyDatetimeCode=false&serverTimezone=Asia/Singapore
        username: stmab
        password: Insteko_666
        hikari:
          pool-name: stmab-hikari-connection-pool
          leak-detection-threshold: 30000
```

**动态数据源配置**

动态确定当前使用的数据源

```groovy
class DynamicDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        String tenantCode = TenantContext.getCurrentTenant()
        ParamAssert.notEmpty(tenantCode, "Tenant code should be provided")
        return tenantCode
    }
}
```

配置多个数据源，并将它们传递给 DynamicDataSource

```groovy
@Configuration
@Order(-1)
class MultiDatasourceConfiguration {
    @Autowired
    private DatasourceConf datasourceConf
    @Autowired
    private TenantConf tenantConf

    @Bean(name = "stmkbDataSource")
    @ConditionalOnProperty(prefix = "spring.datasource.tenants.STMKB", name = "url")
    DataSource stmkbDataSource() {
        HikariConfig hikariConfig = new HikariConfig()

        DatasourceConf.DatasourceConfItem conf = datasourceConf.tenants.get(TenantConst.STMKB_TENANT_CODE)
        hikariConfig.setDriverClassName(conf?.driverClassName)
        hikariConfig.setJdbcUrl(conf?.url)
        hikariConfig.setUsername(conf?.username)
        hikariConfig.setPassword(conf?.password)
        hikariConfig.setPoolName(conf?.hikari?.poolName)
        hikariConfig.setMaximumPoolSize(conf?.hikari?.maximumPoolSize)
        hikariConfig.setTransactionIsolation(conf?.hikari?.transactionIsolation)
        hikariConfig.setMinimumIdle(conf?.hikari?.minimumIdle)
        hikariConfig.setConnectionTimeout(conf?.hikari?.connectionTimeout)
        hikariConfig.setIdleTimeout(conf?.hikari?.idleTimeout)
        hikariConfig.setLeakDetectionThreshold(conf?.hikari?.leakDetectionThreshold)

        return new HikariDataSource(hikariConfig)
    }

    @Bean(name = "stmabDataSource")
    @ConditionalOnProperty(prefix = "spring.datasource.tenants.STMAB", name = "url")
    DataSource stmabDataSource() {
        HikariConfig hikariConfig = new HikariConfig()

        DatasourceConf.DatasourceConfItem conf = datasourceConf.tenants.get(TenantConst.STMAB_TENANT_CODE)
        hikariConfig.setDriverClassName(conf?.driverClassName)
        hikariConfig.setJdbcUrl(conf?.url)
        hikariConfig.setUsername(conf?.username)
        hikariConfig.setPassword(conf?.password)
        hikariConfig.setPoolName(conf?.hikari?.poolName)
        hikariConfig.setMaximumPoolSize(conf?.hikari?.maximumPoolSize)
        hikariConfig.setTransactionIsolation(conf?.hikari?.transactionIsolation)
        hikariConfig.setMinimumIdle(conf?.hikari?.minimumIdle)
        hikariConfig.setConnectionTimeout(conf?.hikari?.connectionTimeout)
        hikariConfig.setIdleTimeout(conf?.hikari?.idleTimeout)
        hikariConfig.setLeakDetectionThreshold(conf?.hikari?.leakDetectionThreshold)

        return new HikariDataSource(hikariConfig)
    }

    @Bean(name = "dynamicDataSource")
    DataSource dataSource(Map<String, DataSource> datasources) {
        DynamicDataSource dynamicDataSource = new DynamicDataSource()

        Map<Object, Object> targetDataSources = new HashMap<>()

        tenantConf.tenantCodes().each { tenantCode ->
            DataSource datasource = datasources.find { beanName, ds ->
                MyStringUtils.startsWithIgnoreCase(beanName, tenantCode)
            }?.getValue()
            if (datasource) {
                targetDataSources.put(tenantCode, datasource)
            }
        }
        dynamicDataSource.setTargetDataSources(targetDataSources)
        return dynamicDataSource
    }
    @Bean
    JdbcTemplate jdbcTemplate(DataSource dynamicDataSource) {
        return new JdbcTemplate(dynamicDataSource)
    }
    // 配置事务管理器
    @Bean
    PlatformTransactionManager transactionManager(DataSource dynamicDataSource) {
        return new DataSourceTransactionManager(dynamicDataSource)
    }
}
```



### **Elasticsearch隔离**

![image-20251105225013755](.\image\image-20251105225013755.png)

1. 数据迁移

a) STMKB index保持不变: bf_***{INDEX}\***

b) STMAB index新增: stmab_bf_***{INDEX}\***

2. 权限控制

a) STMKB user/ STMKB role保持不变, 配置STMKB role仅可操作bf_* index

b) STMAB user/ STMAB role新增: 配置STMAB role仅可操作stmab_bf_* index

3. 数据隔离

a) ElasticSearchConf配置多个数据源

b) EsUtils获取client

```groovy
    static final RestHighLevelClient client() {
        if (!ES_CLIENTS.containsKey(TenantContext.currentTenant)) {
            def nodes = getESNodes()
            synchronized (ESUtils) {
                if (!ES_CLIENTS.containsKey(TenantContext.currentTenant)) {
                    ElasticSearchConf conf = Bean.get(ElasticSearchConf)
                    RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(nodes).setHttpClientConfigCallback(new RestClientBuilder.HttpClientConfigCallback() {
                        @Override
                        HttpAsyncClientBuilder customizeHttpClient(HttpAsyncClientBuilder httpClientBuilder) {
                            def ioReactor = new DefaultConnectingIOReactor()
                            def cm = new PoolingNHttpClientConnectionManager(ioReactor)
                            cm.setMaxTotal(conf.maxPoolSize)
                            if (conf.security.enabled) {
                                final CredentialsProvider credentialProvider = new BasicCredentialsProvider()
                                credentialProvider.setCredentials(
                                        AuthScope.ANY,
                                        new UsernamePasswordCredentials(
                                                conf.security.username,
                                                conf.security.password
                                        )
                                )
                                httpClientBuilder.setDefaultCredentialsProvider(credentialProvider)
                            }
                            return httpClientBuilder.setConnectionManager(cm)
                        }
                    }))
                    ES_CLIENTS.put(TenantContext.currentTenant, client)
                }
            }
        }

        return ES_CLIENTS.get(TenantContext.currentTenant)
    }
```



### **File Storage隔离**

1. 数据迁移

STMKB文件数据: 保持/data/docs_storage挂载不变

STMAB文件数据: 新建文件夹/data/stmab_docs_storage (挂载在新的磁盘)

2. 权限控制

无

3. 数据隔离

a) BlobStorageFolder多租户配置多个folder

b) 写入文件

share/object_storage_file_writer.def根据tenantContext指定folder

c) 获取文件

share/ object_storage_file_read.def根据tenantContext指定folder

d) 删除文件

share/object_storage_file_delete.def根据tenantContext指定folder

```groovy
String getBlobStorageFolder() {
        return blobStorageFolder?.tenants?.get(TenantContext.currentTenant)?.folder
    }
```

### 平台功能

####  BatchJob

a) 区分租户:

i. 扩展@ Scheduled, 以支持多租户

@TenantScheduled

```groovy
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Scheduled
@interface TenantScheduled {
	// 是否为全部tenant执行, 默认为true
    boolean scheduledForAllTenants() default true
 
	// 指定tenant范围
    String[] Tenants() default []
 
    String CRON_DISABLED = "-"
 
    String cron() default ""
 
    String zone() default ""
 
    long fixedDelay() default -1L
 
    String fixedDelayString() default ""
 
    long fixedRate() default -1L
 
    String fixedRateString() default ""
 
    long initialDelay() default -1L
 
    String initialDelayString() default ""
}
```

TenantScheduledAspect

```groovy
@Aspect
@Component
class TenantScheduledAspect {
    @Autowired
    private TenantConf tenantConf
 
@Around("@annotation(tenantScheduled)")
    Object around(ProceedingJoinPoint joinPoint, TenantScheduled tenantScheduled) throws Throwable {
        ParamAssert.isTrue(
tenantScheduled.scheduledForAllTenants() || MyCollectionUtils.isNotNullOrEmpty(tenantScheduled.tenants()?.toList()),
"TenantScheduled needs to specify the tenant."
        )
 
        if (!tenantScheduled.scheduledForAllTenants()) {
            List<String> tenantCodes = tenantConf.list?.findAll { tenant ->
tenantScheduled.tenants().contains(tenant.code)
            }?.collect { tenant -> tenant.code }
 
            ParamAssert.isTrue(
MyCollectionUtils.isNotNullOrEmpty(tenantCodes),
"TenantScheduled needs to specify the valid tenant."
            )
 
            tenantCodes?.each { tenantCode ->
TenantContextUtils.run {
                    return joinPoint.proceed()
}.withItntCode(tenantCode).on(this).go()
            }
        }
 
        tenantConf.list?.each { tenant ->
            TenantContextUtils.run {
                return joinPoint.proceed()
}.withItntCode(tenant?.code).on(this).go()
        }
    }
}
```

i. Job实现多租户串行运行

```groovy
class CampaignBalanceUpdateJob implements Job<JobParams> {
    @Autowired
    JobRunner jobRunner
 
    @Override
    String jobCode() {
        return 'CAMPAIGN_SUM_UPT'
    }
 
    @Override
    String jobName() {
        return 'Campaign Summary Info Update Job'
    }
 
    @Override
    String jobDescription() {
        return "The job is designed to update balance, gwp and cost of campaign after redemption"
    }
 
    @Override
    String jobSchedule() {
        return "Every 30 seconds"
    }
 
    @Override
    void run(JobParams params) {
		  // do something
    }
 
    @TenantScheduled(fixedDelay = MySchedule.TU_30_SECONDS, scheduledForAllTenants = false, tenants = [ProjectConst. STMKB_TENANT_CODE])
    void schedule() {
        if (!EnvUtils.unittest()) {
            jobRunner.runJob(this, null, false)
        }
    }
}
```

#### Task

a) 区分租户

i. TaskProcessingJob通过@TenantScheduled执行,遍历租户执行

ii. 新增Task, 根据TenantContext新增在对应租户数据库

```groovy
Task task = new Task(
		taskType: Bean.get(AutoSubmitProposalTaskProcessor).forTask().code,
		itntCode: policy.itntCode,
		ctntCode: policy.ctntCode,
		bizType: policy.bizType,
		policyId: policy.policyId,	
		transId: policy.transId,
		productCate: policy.productCate,
		productCode: policy.productCode,
		productVersion: policy.productVersion,
		quoteNo: policy.quoteNo,
		policyNo: policy.policyNo,
		generatedAt: MyDateUtils.now(),
		dueAt: MyDateUtils.now(),
		status: PAConst.TASK_STATUS_PENDING,
		i18nTaskName: Messages.i18n()
				.en("Submit proposal ${policy.quoteNo} as it was paid")
				.thai("Submit proposal ${policy.quoteNo} as it was paid"),
		params: MyJsonUtils.toJson([paymentId: payment.paymentId])
)
```

a) 既存数据

识别STMKB和STMAB需要包含哪些Task, 针对Task进行改造

####  多线程

a) 区分租户

i. 任务时在上下文存入tenant_code, 执行异步任务后在上下文清除tenant_code

```groovy
@Bean(destroyMethod = "shutdown", name = GlobalConst.ASYNC_TASK_THREAD_POOL_NAME)
Executor asyncTaskThreadPool() {
    String name = "insmate-async-task-thread-pool"
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor()
    int min = 10
    int max = Runtime.getRuntime().availableProcessors() * 5
    if (max <= min * 2) {
        max = min * 2
    }
    executor.setCorePoolSize(min)
    executor.setMaxPoolSize(max)
    executor.setQueueCapacity(1000)
executor.setThreadNamePrefix("${name}-")
executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy())
    executor.setTaskDecorator(new TaskDecorator() {    
        @Override
        public Runnable decorate(Runnable runnable) {
            return () -> {
                String tenant = TenantContext.currentTenant;
TenantContextHolder.setCurrentTenant(tenant); // 在装饰器中传递租户上下文
                try {
                    runnable.run();
                } finally {
TenantContextHolder.clear(); // 任务完成后清理租户上下文
                }
            };
        }
    })
    executor.initialize()
    return executor
}
```

a) 既存数据

不需要处理

#### OpenAPI

a) 区分tenant

1. Open API定义 参考*Engine, openapi.def按照文件夹隔离租户之间的Open API定义
2. 调用Open API Request header 增加”x-insmate-tenant-code”区分租户

b) 既存数据

不兼容式升级metafin Open API,调用metafin Open API需要在request header增加tenantCode

####  Liquibase

a) 区分租户

i. 为平台定义LiquibaseConf, 全局使用统一的表结构配置文件

```groovy
@Bean
SpringLiquibase liquibase(DataSource tenant1DataSource) {
	SpringLiquibase liquibase = new SpringLiquibase()
	liquibase.setDataSource(tenant1DataSource)
	liquibase.setChangeLog("classpath:db/clients/STMKB/db.changelog-master.xml")  // 指定 changelog 文件
	liquibase.setLiquibaseSchema("")  // 设置 schema，如果需要的话
	liquibase.setShouldRun(false)  // 指定 Liquibase 是否运行
	return liquibase
}
 
@Bean
SpringLiquibase tenant2liquibase(DataSource tenant2DataSource) {
	SpringLiquibase liquibase = new SpringLiquibase();
	liquibase.setDataSource(tenant2DataSource);
	liquibase.setChangeLog("classpath:db/clients/STMKB/db.changelog-master.xml");  // 使用相对路径
	liquibase.setLiquibaseSchema("");  // 设置 schema，如果需要的话
	liquibase.setShouldRun(true);  // 指定 Liquibase 是否运行
	return liquibase;
}
```

a) 既存数据处理

不需要处理