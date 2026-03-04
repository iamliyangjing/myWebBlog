---
description: 本文系统梳理 JVM 调优核心参数，涵盖堆内存、元空间、栈内存配置，Serial、Parallel、CMS、G1、ZGC 等主流垃圾收集器参数详解，GC 日志分析与 OOM 调试技巧，以及生产环境实战配置示例。提供参数速查表与常见问题解决方案，帮助开发者快速定位性能瓶颈、优化 GC 停顿、避免内存溢出。适合 Java 开发者、运维人员面试备战与线上调优参考。
title: JVM 调优参数大全——从内存配置到GC日志，一份超全实战指南
tag:
  - JVM
sidebar: true
comment: true
recommend: 4
---
# JVM 调优参数详解

> 本文档详细讲解 JVM 常用调优参数，包括内存参数、垃圾收集器参数、日志参数等

## 1. 参数分类概览

### 1.1 参数类型

| 类型 | 写法 | 说明 |
|------|------|------|
| **Boolean 类型** | `-XX:+EnableFeature` | 开启某个功能 |
| **Boolean 类型** | `-XX:-DisableFeature` | 关闭某个功能 |
| **K-V 类型** | `-XX:key=value` | 设置参数值 |
| **运行时动态调整** | jinfo -flag | 运行时查看/修改 |

### 1.2 参数查看命令

```bash
# 查看所有参数
java -XX:+PrintFlagsFinal -version

# 查看指定参数
java -XX:+PrintFlagsFinal 2>&1 | grep "G1GC"

# 运行时查看参数
jinfo -flag MetaspaceSize <pid>
jinfo -flag +PrintGCDetails <pid>

# 查看默认 GC
java -XX:+PrintCommandLineFlags -version
```

---

## 2. 内存参数

### 2.1 堆内存参数

```
┌─────────────────────────────────────────────────────────────────┐
│                      JVM 堆内存结构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  -Xms512m         初始堆大小 (Initial Heap)                    │
│  -Xmx2048m        最大堆大小 (Max Heap)                        │
│                   ↑                                            │
│                   ┌────────────────────────────────────┐        │
│                   │              堆                    │        │
│                   │  ┌──────────────┬───────────────┐  │        │
│                   │  │    新生代    │    老年代      │  │        │
│                   │  │    1/3       │    2/3         │  │        │
│                   │  │              │               │  │        │
│                   │  └──────────────┴───────────────┘  │        │
│                   └────────────────────────────────────┘        │
│                                                                 │
│  推荐设置：-Xms 和 -Xmx 设置相同，避免运行时调整                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

| 参数 | 说明 | 示例 |
|------|------|------|
| `-Xms<size>` | 初始堆大小 | `-Xms512m` |
| `-Xmx<size>` | 最大堆大小 | `-Xmx2g` |
| `-Xmn<size>` | 新生代大小 | `-Xmn256m` |
| `-XX:NewRatio=<ratio>` | 老年代:新生代比例 | `-XX:NewRatio=2`（老年代:新生代=2:1） |
| `-XX:SurvivorRatio=<ratio>` | Eden:Survivor 比例 | `-XX:SurvivorRatio=8`（Eden:S0:S1=8:1:1） |

#### 常用配置示例

```bash
# 固定堆大小 2GB
java -Xms2g -Xmx2g MyApp

# 新生代 512MB，老年代 1.5GB（NewRatio=2）
java -Xms2g -Xmx2g -Xmn512m MyApp

# 自定义 Survivor 比例
java -Xms2g -Xmx2g -XX:SurvivorRatio=4
# Survivor = 512M / (4+2) = 85M
# Eden = 85M * 4 = 340M
```

### 2.2 方法区/元空间参数

| 参数 | 说明 | 示例 |
|------|------|------|
| `-XX:MetaspaceSize=<size>` | 元空间初始大小 | `-XX:MetaspaceSize=256m` |
| `-XX:MaxMetaspaceSize=<size>` | 元空间最大大小 | `-XX:MaxMetaspaceSize=512m` |
| `-XX:PermSize=<size>` | 永久代初始大小（JDK 7） | `-XX:PermSize=128m` |
| `-XX:MaxPermSize=<size>` | 永久代最大大小（JDK 7） | `-XX:MaxPermSize=256m` |

#### JDK 8+ 最佳实践

```bash
# 推荐：只设置最大元空间
java -XX:MaxMetaspaceSize=512m MyApp

# 不推荐设置初始值，让 JVM 自动调整
```

### 2.3 栈内存参数

| 参数 | 说明 | 示例 |
|------|------|------|
| `-Xss<size>` | 每个线程的栈大小 | `-Xss1m` |
| `-XX:ThreadStackSize=<size>` | 同上 | `-XX:ThreadStackSize=1024` |

#### 栈大小选择

| 应用类型 | 建议栈大小 |
|----------|------------|
| 普通 Java 应用 | 1MB（默认） |
| 递归深度大 | 2MB |
| 线程数多 | 减小栈大小 |

```bash
# 减少栈大小，创建更多线程
java -Xss256k MyApp
```

---

## 3. 垃圾收集器参数

### 3.1 收集器选择参数

```
┌─────────────────────────────────────────────────────────────────┐
│                      收集器选择                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Serial 收集器                                                  │
│    -XX:+UseSerialGC       Serial + Serial Old                   │
│                                                                 │
│  ParNew 收集器                                                  │
│    -XX:+UseParNewGC      ParNew + Serial Old                   │
│                                                                 │
│  Parallel Scavenge 收集器                                       │
│    -XX:+UseParallelGC    Parallel Scavenge + Serial Old        │
│    -XX:+UseParallelOldGC  Parallel Scavenge + Parallel Old      │
│                                                                 │
│  CMS 收集器                                                     │
│    -XX:+UseConcMarkSweepGC  ParNew + CMS + Serial Old          │
│                                                                 │
│  G1 收集器                                                     │
│    -XX:+UseG1GC                                                │
│                                                                 │
│  ZGC 收集器 (JDK 11+)                                          │
│    -XX:+UseZGC                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Serial 收集器参数

| 参数 | 说明 |
|------|------|
| `-XX:+UseSerialGC` | 启用 Serial 收集器 |

### 3.3 ParNew 收集器参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `-XX:+UseParNewGC` | 启用 ParNew 收集器 | |
| `-XX:ParallelGCThreads` | GC 线程数 | CPU 核心数 |
| `-XX:MaxGCPauseMillis` | 最大停顿时间目标 | 无限制 |

### 3.4 Parallel Scavenge 收集器参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `-XX:+UseParallelGC` | 启用 Parallel Scavenge | |
| `-XX:+UseParallelOldGC` | 使用 Parallel Old | |
| `-XX:ParallelGCThreads` | GC 线程数 | CPU 核心数 |
| `-XX:MaxGCPauseMillis` | 最大停顿时间（期望值） | 200ms |
| `-XX:GCTimeRatio` | GC 时间占比（吞吐量=1/(1+GCTimeRatio)） | 99 |
| `-XX:+UseAdaptiveSizePolicy` | 自动调整各区域大小 | 开启 |

#### 吞吐量配置示例

```bash
# 目标吞吐量 99%（GC 时间占比 1%）
java -XX:GCTimeRatio=99 -XX:MaxGCPauseMillis=200 MyApp
# 吞吐量 = 99 / (99 + 1) = 99%
```

### 3.5 CMS 收集器参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `-XX:+UseConcMarkSweepGC` | 启用 CMS | |
| `-XX:CMSInitiatingOccupancyFraction` | 老年代使用率触发 CMS | 68% |
| `-XX:+UseCMSCompactAtFullCollection` | FullGC 时整理碎片 | 开启 |
| `-XX:CMSFullGCsBeforeCompaction` | 多少次 FullGC 后整理 | 0（每次） |
| `-XX:+UseCMSInitiatingOccupancyOnly` | 只按设定阈值触发 | 关闭 |
| `-XX:CMSScheduleRemarkEdenSizeThreshold` | 重新标记前 Eden 大小 | 2MB |
| `-XX:CMSScheduleRemark EdenPenetration` | 重新标记 Eden 使用率 | 50% |

#### CMS 配置示例

```bash
# CMS 配置
java -XX:+UseConcMarkSweepGC \
     -XX:CMSInitiatingOccupancyFraction=75 \
     -XX:+UseCMSCompactAtFullCollection \
     -XX:CMSFullGCsBeforeCompaction=3 \
     MyApp
```

### 3.6 G1 收集器参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `-XX:+UseG1GC` | 启用 G1 | |
| `-XX:MaxGCPauseMillis` | 目标停顿时间 | 200ms |
| `-XX:G1HeapRegionSize` | Region 大小 | 1MB~32MB |
| `-XX:InitiatingHeapOccupancyPercent` | 触发并发标记阈值 | 45% |
| `-XX:G1NewSizePercent` | 年轻代最小比例 | 5% |
| `-XX:G1MaxNewSizePercent` | 年轻代最大比例 | 60% |
| `-XX:G1ReservePercent` | 保留内存比例 | 10% |
| `-XX:ParallelGCThreads` | 并行 GC 线程数 | |
| `-XX:ConcGCThreads` | 并发 GC 线程数 | |

#### G1 配置示例

```bash
# G1 配置
java -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=100 \
     -XX:G1HeapRegionSize=16m \
     -XX:InitiatingHeapOccupancyPercent=45 \
     -XX:G1ReservePercent=10 \
     MyApp
```

### 3.7 ZGC 收集器参数 (JDK 11+)

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `-XX:+UseZGC` | 启用 ZGC | |
| `-XX:ConcGCThreads` | 并发 GC 线程数 | 自动 |
| `-XX:ParallelGCThreads` | GC 线程数 | 自动 |
| `-XX:ZHeapSize` | 堆大小 | 堆大小 |

---

## 4. GC 日志参数

### 4.1 常用日志参数

```
┌─────────────────────────────────────────────────────────────────┐
│                      GC 日志参数                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  基本日志                                                      │
│    -verbose:gc                  打印 GC 信息                    │
│    -XX:+PrintGC                 打印 GC 信息                    │
│    -XX:+PrintGCDetails         打印 GC 详细信息                 │
│    -XX:+PrintGCTimeStamps      打印 GC 发生的时间戳             │
│    -XX:+PrintGCDateStamps     打印 GC 发生的日期时间            │
│                                                                 │
│  输出到文件                                                   │
│    -Xloggc:<filename>          GC 日志输出到文件                │
│                                                                 │
│  GC 日志轮转                                                   │
│    -XX:+UseGCLogFileRotation   启用日志轮转                     │
│    -XX:NumberOfGCLogFiles=3    日志文件数量                    │
│    -XX:GCLogFileSize=8m        单个日志文件大小                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 日志配置示例

```bash
# 简单 GC 日志
java -XX:+PrintGC -Xms512m -Xmx512m MyApp

# 详细 GC 日志
java -XX:+PrintGCDetails \
     -XX:+PrintGCTimeStamps \
     -XX:+PrintGCDateStamps \
     -Xloggc:./logs/gc.log \
     MyApp

# 日志轮转配置
java -XX:+UseG1GC \
     -Xlog:gc*:file=./logs/gc.log:time,uptime,level,tags:filecount=5,filesize=10m \
     MyApp
```

### 4.3 JDK 9+ 统一日志配置

```bash
# JDK 9+ 推荐写法
java -Xlog:gc*=info:file=./logs/gc.log:time,uptime:filecount=10,filesize=20m MyApp

# 含义：
# gc*=info     - 所有 gc 标签的日志级别为 info
# file=        - 输出到文件
# time,uptime  - 包含时间和运行时间
# filecount    - 最多保留 10 个文件
# filesize     - 单个文件最大 20MB
```

#### 日志级别

| 级别 | 说明 |
|------|------|
| off | 关闭日志 |
| severe | 严重 |
| warning | 警告 |
| info | 信息 |
| debug | 调试 |
| trace | 跟踪 |

---

## 5. OOM 调试参数

### 5.1 堆转储参数

```
┌─────────────────────────────────────────────────────────────────┐
│                      OOM 调试参数                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  堆转储（Heap Dump）                                           │
│    -XX:+HeapDumpOnOutOfMemoryError   OOM 时生成堆转储           │
│    -XX:HeapDumpPath=<path>           堆转储保存路径             │
│                                                                 │
│  示例：                                                         │
│    java -XX:+HeapDumpOnOutOfMemoryError                         │
│          -XX:HeapDumpPath=/tmp/heapdump.hprof                  │
│          -Xms100m -Xmx100m                                     │
│          -XX:+HeapDumpOnOutOfMemoryError MyApp                  │
│                                                                 │
│  OOM 时执行脚本                                                │
│    -XX:OnOutOfMemoryError=<command>   OOM 时执行命令           │
│                                                                 │
│  示例：                                                         │
│    -XX:OnOutOfMemoryError="sh ~/restart.sh"                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 完整配置示例

```bash
# 生产环境推荐配置
java -server \
     -Xms4g -Xmx4g \
     -Xmn2g \
     -XX:MetaspaceSize=256m \
     -XX:MaxMetaspaceSize=512m \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/oom.hprof \
     -Xlog:gc*:file=/var/log/gc.log:time,uptime:filecount=10,filesize=50m \
     -XX:+PrintGCDetails \
     -XX:+PrintGCDateStamps \
     MyApp
```

---

## 6. 性能调优参数

### 6.1 对象晋升参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `-XX:MaxTenuringThreshold=<age>` | 对象进入老年代年龄 | 15 |
| `-XX:TargetSurvivorRatio=<percent>` | Survivor 区目标使用率 | 50% |
| `-XX:PretenureSizeThreshold=<size>` | 大对象直接进入老年代阈值 | 0（不启用） |

#### 配置示例

```bash
# 大对象直接进老年代
java -XX:PretenureSizeThreshold=2m MyApp

# 调整对象年龄
java -XX:MaxTenuringThreshold=10 MyApp
```

### 6.2 软引用/弱引用参数

| 参数 | 说明 |
|------|------|
| `-XX:SoftRefLRUPolicyMSPerMB` | 软引用存活时间（每 MB 空闲时间） |

### 6.3 JIT 编译器参数

```
┌─────────────────────────────────────────────────────────────────┐
│                      JIT 编译器参数                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  编译相关                                                      │
│    -server                服务端模式（Server VM）               │
│    -client                客户端模式（Client VM）               │
│    -XX:+TieredCompilation  启用分层编译（JDK 8+）               │
│    -XX:CompileThreshold    触发 JIT 编译的调用次数             │
│                                                                 │
│  方法内联                                                     │
│    -XX:MaxInlineSize=<size>       最大内联方法字节码大小        │
│    -XX:FreqInlineSize=<size>     热点方法内联大小              │
│                                                                 │
│  示例：                                                         │
│    -XX:+TieredCompilation                                    │
│    -XX:CompileThreshold=10000                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.4 其他性能参数

| 参数 | 说明 |
|------|------|
| `-XX:+UseStringDeduplication` | 字符串去重（JDK 8+） |
| `-XX:+OptimizeStringConcat` | 优化字符串拼接 |
| `-XX:+UseCompressedOops` | 压缩对象指针（64位 JVM） |
| `-XX:+UseCompressedClassPointers` | 压缩类指针 |

---

## 7. 常用参数速查表

### 7.1 内存参数速查

```bash
# 堆内存
-Xms512m              # 初始堆 512MB
-Xmx2g                # 最大堆 2GB
-Xmn256m              # 新生代 256MB

# 元空间
-XX:MetaspaceSize=256m    # 元空间初始
-XX:MaxMetaspaceSize=512m # 元空间最大

# 栈
-Xss1m                 # 线程栈 1MB

# 比例
-XX:NewRatio=2         # 老年代:新生代 = 2:1
-XX:SurvivorRatio=8    # Eden:Survivor = 8:1
```

### 7.2 GC 参数速查

```bash
# 收集器选择
-XX:+UseSerialGC           # Serial
-XX:+UseParNewGC           # ParNew
-XX:+UseParallelGC         # Parallel
-XX:+UseConcMarkSweepGC    # CMS
-XX:+UseG1GC               # G1
-XX:+UseZGC                # ZGC

# CMS 特定
-XX:CMSInitiatingOccupancyFraction=75

# G1 特定
-XX:MaxGCPauseMillis=200
```

### 7.3 日志参数速查

```bash
# GC 日志
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-Xloggc:./gc.log

# OOM
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heap.hprof
```

---

## 8. 实际配置示例

### 8.1 小型应用（堆 512MB）

```bash
java -Xms512m -Xmx512m \
     -XX:+UseSerialGC \
     -XX:+PrintGCDetails \
     -Xloggc:./gc.log \
     MyApp
```

### 8.2 中型应用（堆 2GB，G1）

```bash
java -server \
     -Xms2g -Xmx2g \
     -Xmn1g \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:InitiatingHeapOccupancyPercent=45 \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/oom.hprof \
     -Xlog:gc*=info:file=/var/log/gc.log:time,uptime:filecount=5,filesize=10m \
     MyApp
```

### 8.3 大型应用（堆 8GB，吞吐量优先）

```bash
java -server \
     -Xms8g -Xmx8g \
     -Xmn3g \
     -XX:+UseParallelOldGC \
     -XX:ParallelGCThreads=8 \
     -XX:+UseAdaptiveSizePolicy \
     -XX:MaxGCPauseMillis=500 \
     -XX:GCTimeRatio=19 \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/oom.hprof \
     -Xlog:gc*=info:file=/var/log/gc.log:time,uptime:filecount=10,filesize=50m \
     MyApp
```

---

## 9. 常见问题与解决方案

### 9.1 参数设置原则

| 场景 | 建议 |
|------|------|
| 堆大小 | `-Xms` 和 `-Xmx` 设为相同，避免运行时调整 |
| 新生代 | 建议堆的 1/3 ~ 1/2 |
| CMS | 预留足够空间给浮动垃圾 |
| G1 | 合理设置停顿时间目标 |
| 日志 | 生产环境务必开启 GC 日志 |

### 9.2 常见问题

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| Full GC 频繁 | 晋升太快、内存不足 | 增大新生代、调低年龄阈值 |
| 内存持续增长 | 内存泄漏 | 使用 MAT 分析 |
| GC 停顿长 | 大对象、碎片化 | 选择合适收集器、调优参数 |
| Metaspace OOM | 类加载过多 | 增大 MetaspaceSize |

---

## 10. 总结

### 核心参数汇总

```
┌─────────────────────────────────────────────────────────────────┐
│                    必知必会参数                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  内存：-Xms -Xmx -Xmn -XX:MetaspaceSize                        │
│                                                                 │
│  收集器：-XX:+UseG1GC -XX:+UseSerialGC 等                       │
│                                                                 │
│  日志：-XX:+PrintGCDetails -Xloggc:                            │
│                                                                 │
│  调试：-XX:+HeapDumpOnOutOfMemoryError                         │
│                                                                 │
│  停顿时间：-XX:MaxGCPauseMillis                                 │
│                                                                 │
│  吞吐量：-XX:GCTimeRatio                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

> **下一章**：[模块五：内存模型(JMM)](./05-内存模型.md)
