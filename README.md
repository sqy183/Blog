# Android 资深工程师知识库

> 面向 5-10 年 Android 开发工程师的系统性技术知识库。以 AOSP 源码级深度解析核心知识点，兼顾面试准备与日常查阅。

---

## 知识体系

本知识库按照 [技能图谱](Android/技能图谱.md) 大纲组织，覆盖以下核心领域：

| 章节 | 方向 | 目录 |
|------|------|------|
| 一 | Java / Kotlin 语言基础 | [Java与Kotlin基础](Android/Java与Kotlin基础/) |
| 二 | Android Framework 层 | [Framework](Android/Framework/) |
| 三 | UI 与渲染体系 | [UI与渲染](Android/UI与渲染/) |
| 四 | Jetpack 与架构 | [Jetpack与架构](Android/Jetpack与架构/) |
| 五 | 性能优化 | [性能优化](Android/性能优化/) |
| 六 | 开源框架原理 | [开源框架原理](Android/开源框架原理/) |
| 七 | 跨平台与新技术 | 待建设 |
| 八 | 工程化与 DevOps | 待建设 |
| 九 | Android 系统底层 | 待建设 |
| 十 | 端侧 AI 与大模型 | 待建设 |
| 十一 | 计算机基础 | [计算机基础](计算机基础/) |

## 文档索引

### Android 核心

| 文档 | 路径 | 核心内容 |
|------|------|---------|
| App 冷启动流程 | [app启动流程.md](Android/Framework/app启动流程.md) | 五阶段全流程、Zygote fork 机制、启动度量、优化实战、10 道面试题 |
| 消息机制 | [消息机制.md](Android/Framework/消息机制.md) | Handler/Looper/MessageQueue 源码分析、同步屏障、IdleHandler、设计哲学、面试题 |
| 类加载机制 | [类加载机制.md](Android/Java与Kotlin基础/类加载机制.md) | JVM 类加载、双亲委派、Android ClassLoader、dexElements 热修复原理、7 道面试题 |
| 并发编程与线程安全 | [并发编程与线程安全.md](Android/Java与Kotlin基础/并发编程与线程安全.md) | JMM、锁升级、AQS、CAS、ThreadLocal、线程池、Kotlin 协程、面试题 |
| Android 页面绘制 | [Android页面绘制.md](Android/UI与渲染/Android页面绘制.md) | View 绘制三大流程、类持有关系链、VSync 渲染流水线、BufferQueue 跨进程机制、面试题 |
| Activity 生命周期与启动模式 | [Activity生命周期与启动模式.md](Android/Framework/Activity生命周期与启动模式.md) | 七大回调与状态机、边界场景、五种启动模式、Task 与 Intent Flags、10 道面试题 |
| 内存泄漏与内存优化 | [内存泄漏与内存优化.md](Android/性能优化/内存泄漏与内存优化.md) | 内存管理机制、7 种泄漏场景、LeakCanary 原理、Bitmap 优化、OOM 治理、10 道面试题 |
| WMS 与 PMS 核心原理 | [WMS与PMS核心原理.md](Android/Framework/WMS与PMS核心原理.md) | 窗口类型与层级、窗口添加流程、Token 机制、APK 安装、权限管理、Intent 解析、10 道面试题 |

### Jetpack 与架构

| 文档 | 路径 | 核心内容 |
|------|------|---------|
| ViewModel 数据保持原理 | [ViewModel数据保持原理.md](Android/Jetpack与架构/ViewModel数据保持原理.md) | NonConfigurationInstances 存活机制、ViewModelStore/ViewModelProvider 源码、Factory 体系、SavedStateHandle、Fragment 共享、8 道面试题 |

### 开源框架原理

| 文档 | 路径 | 核心内容 |
|------|------|---------|
| OkHttp 拦截器链原理 | [OkHttp拦截器链原理.md](Android/开源框架原理/OkHttp拦截器链原理.md) | 责任链模式、5 大内置拦截器、连接池复用、Application vs Network Interceptor、8 道面试题 |

### 计算机基础

| 文档 | 路径 |
|------|------|
| 计算机网络 | [计算机网络概述.md](计算机基础/计算机网络/计算机网络概述.md) |
| 计算机操作系统 | [计算机操作系统概述.md](计算机基础/计算机操作系统/计算机操作系统概述.md) |
| 计算机组成 | [计算机组成概述.md](计算机基础/计算机组成/计算机组成概述.md) |
| 数据库 | [数据库概述.md](计算机基础/数据库/数据库概述.md) |

## 建设进度

详见 [PROGRESS.md](PROGRESS.md)。当前已完成 Framework 层和语言基础的重点文档，持续建设中。

## 写作标准

- 以资深 Android 工程师视角撰写，兼顾深度与可读性
- 源码级分析标注 AOSP 版本，关键调用链配有流程图
- 每篇文档附带高频面试题与详细解答
- 完整的写作规范见 [CLAUDE.md](CLAUDE.md)
