---
description: Spring IOC概念及使用方法
title: Spring IOC 详解
tag:
  - Spring
  - 框架
sidebar: true
comment: true
recommend: 1
---

## 一、IOC是什么

IOC（Inverse of Control:控制反转）是一种**设计思想**，就是 将**原本在程序中手动创建对象的控制权，交由Spring框架来管理**。 IOC 在其他语言中也有应用，并非 Spirng 特有。 IoC 容器是 Spring 用来实现 IOC的载体， IOC容器实际上就是个Map（key，value）,Map 中存放的是各种对象。将**对象之间的相互依赖关系交给 IoC 容器来管理，并由 IoC 容器完成对象的注入**。这样可以很大程度上简化应用的开发，把应用从复杂的[依赖关系](https://so.csdn.net/so/search?q=依赖关系&spm=1001.2101.3001.7020)中解放出来。 IOC容器就像是一个工厂一样，当我们需要创建一个对象的时候，只需要配置好配置文件/注解即可，完全不用考虑对象是如何被创建出来的。

在实际项目中一个 Service 类可能有几百甚至上千个类作为它的底层，假如我们需要实例化这个 Service，你可能要每次都要搞清这个 Service 所有底层类的构造函数，这可能会把人逼疯。如果利用 IoC 的话，你只需要配置好，然后在需要的地方引用就行了，这大大增加了项目的可维护性且降低了开发难度。

Spring 时代我们一般通过 XML 文件来配置 Bean，后来开发人员觉得 XML 文件来配置不太好，于是 SpringBoot 注解配置就慢慢开始流行起来。

## 二、Spring 实现IOC有哪些方式

### DI 和 IOC的概念

DI（dependency injection），依赖注入。DI就是在指IOC容器内实现的将依赖对象注入的概念。

IOC（inverse of control），将程序中创建对象的控制权交给容器去做，而不是由开发人员来做。就是控制反转，由IOC容器来管理对象的创建，消耗。

**关系：**<font color='red'>IOC就是容器，DI就是注入这一行为</font>，那么DI确实就是IOC的具体功能的实现。而IOC则是DI发挥的平台和空间。

实现IOC的两种方式：

- 利用<font color='red'>**XML形式**</font>实现IOC
- 使用<font color='red'>**注解形式**</font>实现IOC

### 四种依赖注入的方式

Spring 依赖注入有四种方式：属性（setter）注入，构造器注入，静态工厂方法注入，实例工厂方法注入。最常见的是属性注入和构造器注入。下面对这几种注入方法举例说明：

#### **Setter 注入**

通过构造器传递依赖对象，Spring 容器在创建 Bean 时注入依赖。
示例：

```java
public class MyService {
    private MyRepository repository;

    public void setRepository(MyRepository repository) {
        this.repository = repository;
    }
}
```

#### **构造器注入**

通过构造器传递依赖对象，Spring 容器在创建 Bean 时注入依赖。
示例：

```java
public class MyService {
    private final MyRepository repository;

    public MyService(MyRepository repository) {
        this.repository = repository;
    }
}
```

#### **字段注入**

使用 `@Autowired` 或 `@Resource` 注解直接注入字段，Spring 容器自动完成注入。
示例：

```java
public class MyService {
    @Autowired
    private MyRepository repository;
}
```

#### **方法注入**

通过任意方法注入依赖，Spring 容器调用该方法完成注入。
示例：

```java
public class MyService {
    private MyRepository repository;

    @Autowired
    public void setupRepository(MyRepository repository) {
        this.repository = repository;
    }
}
```

#### **接口注入**

实现特定接口，Spring 容器通过接口方法注入依赖。
示例：

```java
public interface RepositoryAware {
    void setRepository(MyRepository repository);
}

public class MyService implements RepositoryAware {
    private MyRepository repository;

    @Override
    public void setRepository(MyRepository repository) {
        this.repository = repository;
    }
}
```
### 利用注解实现IOC

- @Component ：通用的注解，可标注任意类为 Spring 组件。如果一个 Bean 不知道属于哪个层，可以使用@Component 注解标注。
- @Repository : 对应持久层即 Dao 层，主要用于数据库相关操作。
- @Service : 对应服务层，主要涉及一些复杂的逻辑，需要用到 Dao 层。
- @Controller : 对应 Spring MVC 控制层，主要用户接受用户请求并调用 Service 层返回数据给前端页面。

### 总结

- **构造器注入**：推荐使用，确保依赖不可变，避免空指针异常。
- **Setter 注入**：适合可选依赖。
- **字段注入**：简洁，但不易测试且可能隐藏依赖关系。
- **方法注入**：灵活，适用于复杂场景。
- **接口注入**：较少使用，适用于特定需求。

根据具体场景选择合适的注入方式。