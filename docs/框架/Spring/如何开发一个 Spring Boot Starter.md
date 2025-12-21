# 从零开发一个 Spring Boot Starter

> 本文将通过 **完整可运行 Demo**，手把手教你如何从 0 到 1 开发一个 **Spring Boot Starter**。适合已经使用过 Spring Boot、希望提升框架设计能力的开发者。

------

## 一、什么是 Spring Boot Starter

### 1. Starter 的本质

**Spring Boot Starter 本质上就是一个 Maven 依赖聚合包 + 自动配置模块**，用于：

- 封装公共能力
- 减少重复配置
- 实现“引入依赖即可用”

例如：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

背后做了三件事：

1. 引入相关依赖（Spring MVC、Jackson、Tomcat）
2. 提供自动配置类（AutoConfiguration）
3. 根据条件自动生效（@Conditional）

------

### 2. Starter 的典型结构

```text
xxx-spring-boot-starter
├── xxx-spring-boot-autoconfigure
│   └── 自动配置代码
└── pom.xml（只做依赖聚合）
```

**最佳实践：starter 与 autoconfigure 分离**（Spring 官方也是这么做的）。

------

## 二、Demo 目标说明

我们来实现一个简单但完整的 Starter：

👉 **hello-spring-boot-starter**

功能：

- 提供一个 `HelloService`
- 支持在 `application.yml` 中配置：

```yaml
hello:
  prefix: "Hello"
  suffix: "!"
```

使用效果：

```java
@Autowired
private HelloService helloService;

helloService.sayHello("Spring Boot");
// 输出：Hello Spring Boot!
```

------

## 三、创建父工程

### 1. Maven 父工程

```xml
<groupId>com.example</groupId>
<artifactId>hello-spring-boot</artifactId>
<version>1.0.0</version>
<packaging>pom</packaging>

<modules>
    <module>hello-spring-boot-starter</module>
    <module>hello-spring-boot-autoconfigure</module>
</modules>
```

------

## 四、实现 AutoConfigure 模块（核心）

### 1. 创建 hello-spring-boot-autoconfigure

```xml
<artifactId>hello-spring-boot-autoconfigure</artifactId>
```

依赖：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-autoconfigure</artifactId>
</dependency>

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-configuration-processor</artifactId>
  <optional>true</optional>
</dependency>
```

------

### 2. 编写配置属性类

```java
@ConfigurationProperties(prefix = "hello")
public class HelloProperties {

    private String prefix = "Hello";
    private String suffix = "";

    // getter / setter
}
```

------

### 3. 核心业务类

```java
public class HelloService {

    private final HelloProperties properties;

    public HelloService(HelloProperties properties) {
        this.properties = properties;
    }

    public String sayHello(String name) {
        return properties.getPrefix() + " " + name + properties.getSuffix();
    }
}
```

------

### 4. 自动配置类（最关键）

```java
@Configuration
@EnableConfigurationProperties(HelloProperties.class)
@ConditionalOnClass(HelloService.class)
public class HelloAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public HelloService helloService(HelloProperties properties) {
        return new HelloService(properties);
    }
}
```

📌 说明：

- `@ConditionalOnClass`：类存在才加载
- `@ConditionalOnMissingBean`：允许用户自定义覆盖
- Starter 一定要 **给用户留扩展点**

------

### 5. 注册自动配置（Spring Boot 2.x / 3.x）

#### Spring Boot 2.x

```text
META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.hello.autoconfigure.HelloAutoConfiguration
```

#### Spring Boot 3.x（推荐）

```text
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.hello.autoconfigure.HelloAutoConfiguration
```

------

## 五、实现 Starter 模块（依赖聚合）

### 1. 创建 hello-spring-boot-starter

```xml
<artifactId>hello-spring-boot-starter</artifactId>
```

### 2. 只做依赖引入

```xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>hello-spring-boot-autoconfigure</artifactId>
</dependency>
```

❌ 不写任何 Java 代码

------

## 六、使用 Starter（验证效果）

### 1. 新建 Spring Boot 应用

```xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>hello-spring-boot-starter</artifactId>
  <version>1.0.0</version>
</dependency>
```

------

### 2. application.yml

```yaml
hello:
  prefix: "Hi"
  suffix: " ~"
```

------

### 3. 使用

```java
@RestController
public class TestController {

    @Autowired
    private HelloService helloService;

    @GetMapping("/hello")
    public String hello() {
        return helloService.sayHello("Starter");
    }
}
```

访问：

```
GET /hello
→ Hi Starter ~
```

🎉 Starter 生效！

------

## 七、Starter 设计最佳实践

### 1. 一定要用 Conditional

常用条件：

- `@ConditionalOnClass`
- `@ConditionalOnMissingBean`
- `@ConditionalOnProperty`

------

### 2. Starter ≠ 业务代码

✅ Starter：

- 通用能力
- 框架级封装

❌ Starter：

- 强业务逻辑
- 写死的配置

------

### 3. 给用户足够的覆盖能力

```java
@ConditionalOnMissingBean
```

这是 Starter 能否被接受的关键。

------

## 八、常见问题

### Q1：为什么 starter 和 autoconfigure 要拆？

- 避免用户误依赖
- 解耦自动配置与依赖管理
- 符合 Spring 官方规范

------

### Q2：Starter 适合哪些场景？

- 公司内部公共组件
- 中间件 SDK
- 技术平台能力封装

------

## 九、自动装配原理（面试高频）

这一节是**面试 + 技术影响力**的核心内容，很多面试官会从 Starter 直接追问到 **Spring Boot 自动装配机制**。

------

### 1. 自动装配解决了什么问题？

在没有自动装配之前，Spring 使用方式是：

```java
@Configuration
public class XxxConfig {

    @Bean
    public HelloService helloService() {
        return new HelloService();
    }
}
```

问题在于：

- 每个项目都要写一遍配置
- 第三方组件无法做到“开箱即用”
- 配置侵入业务代码

**自动装配的目标只有一个：**

> 👉 框架根据“环境 + 条件”，自动帮你把 Bean 装配好

------

### 2. @SpringBootApplication 做了什么？（必考）

```java
@SpringBootApplication
```

等价于：

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
```

其中：

- `@ComponentScan`：扫描业务 Bean
- `@SpringBootConfiguration`：本质是 `@Configuration`
- **`@EnableAutoConfiguration`：自动装配的入口（重点）**

------

### 3. 自动装配的核心链路（一定要能说清）

整体流程如下：

```text
@SpringBootApplication
        ↓
@EnableAutoConfiguration
        ↓
AutoConfigurationImportSelector
        ↓
加载 META-INF 下的自动配置声明文件
        ↓
按条件(@Conditional)决定是否生效
        ↓
向 IOC 容器注册 Bean
```

------

### 4. 自动配置类是如何被发现的？

#### Spring Boot 2.x

通过：

```text
META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.example.hello.autoconfigure.HelloAutoConfiguration
```

Spring Boot 启动时会：

- 扫描所有依赖 jar
- 汇总 `spring.factories`
- 加载所有 AutoConfiguration 类

------

#### Spring Boot 3.x（新机制，面试加分）

```text
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

特点：

- 不再 key-value
- 启动更快
- 配置更清晰

------

### 5. @Conditional 是自动装配的“灵魂”

如果没有条件装配：

> 所有 Starter 一起加载 = 灾难 💥

常见条件注解：

```java
@ConditionalOnClass
@ConditionalOnMissingBean
@ConditionalOnProperty
@ConditionalOnBean
```

举例说明（非常适合面试）：

```java
@Bean
@ConditionalOnMissingBean
public HelloService helloService() {
    return new HelloService();
}
```

含义：

> 如果用户自己定义了 HelloService，Starter 自动让位

📌 **这体现的是 Spring Boot 的设计哲学：约定优于配置，但不强制。**

------

### 6. Starter 为什么一定要允许覆盖？（面试常问）

如果 Starter：

- 不允许用户自定义 Bean
- 强行接管配置

结果就是：

❌ Starter 很“好用”，但没人敢用

**一个合格的 Starter：**

- 默认好用
- 但永远可以被替换

------

## 十、面试视角下的 Starter 设计总结

你在面试中可以这样总结：

> Spring Boot Starter 本质上是：
>
> - 以 `@EnableAutoConfiguration` 为入口
> - 通过 SPI 机制发现自动配置类
> - 使用 `@Conditional` 决定是否装配 Bean
> - 最终实现“依赖即配置”的框架能力封装

如果你能把这条链路讲清楚：

👉 **说明你理解的不是 API，而是 Spring Boot 的设计本身。**

------

## 十一、总结（技术影响力版本）

> 初级开发者使用 Spring Boot
>
> 中级开发者配置 Spring Boot
>
> **高级开发者设计 Spring Boot Starter**

当你开始写 Starter，说明你的视角已经从：

> “怎么用框架”

转变为：

> “框架应该怎么被别人使用”

这正是**技术影响力**的起点。