# ART 虚拟机原理

## 一、概述

Android 应用用 Java/Kotlin 编写，但从未运行在标准 JVM（HotSpot）上。Android 有自己的虚拟机——从早期的 **Dalvik** 到现在的 **ART（Android Runtime）**。理解 ART 的工作原理，是深入性能优化、热修复、插件化、崩溃分析等高级话题的基础。

> ART 的核心设计目标：**在移动设备有限的内存和电量约束下，尽可能接近（甚至超越）服务端 JVM 的执行效率**。为此它在字节码格式、编译策略、GC 算法上都做了大量针对性设计。

本文覆盖以下核心话题：
1. 为什么 Android 不用标准 JVM
2. Dalvik → ART 的演进历程
3. Dex 字节码与 Java 字节码的本质差异
4. ART 编译策略的三代演进（AOT → JIT → 混合编译）
5. dex2oat 编译管线
6. ART 的 GC 机制
7. 运行时文件体系（oat/vdex/art/odex）
8. 运行时优化技术

## 二、为什么 Android 不用标准 JVM

### 2.1 历史背景

2005 年 Google 收购 Android Inc. 时面临一个关键决策：用什么语言和运行时？最终选择了 **Java 语言 + 自研虚拟机**，原因：

- **语言生态**：Java 开发者基数最大，降低开发门槛
- **许可证规避**：当时 Sun（后来的 Oracle）的 JVM 实现受 GPL 许可约束，Google 不想受限于此
- **硬件约束**：2008 年的手机只有 192MB RAM、528MHz CPU，标准 JVM（HotSpot）太重了

于是 Google 的 Dan Bornstein 设计了 **Dalvik VM**，核心设计决策：

| 决策 | 标准 JVM | Dalvik/ART | 原因 |
|------|---------|------------|------|
| 指令架构 | 基于栈 | 基于寄存器 | 减少指令数，解释执行更快 |
| 文件格式 | .class（每类一文件） | .dex（多类合一） | 共享常量池，减少内存占用 |
| 进程模型 | 每个应用独立 JVM 实例 | Zygote fork 共享内存 | 极大加速应用启动，节省内存 |
| GC | 分代收集为主 | 针对移动场景优化 | 减少 GC 停顿，避免界面卡顿 |

### 2.2 Zygote 共享模型

这是 Android 虚拟机最独特的设计之一，标准 JVM 没有这个概念：

```
系统启动时:
  Init 进程 → 启动 Zygote 进程
    Zygote 做了什么：
    ① 创建第一个 ART 虚拟机实例
    ② 预加载 ~15000 个常用类（framework 类、Android SDK 类）
    ③ 预加载共享资源（主题、布局等）
    ④ 进入 socket 监听，等待请求

每次启动新应用:
  AMS → 通过 socket 请求 Zygote
    Zygote 调用 fork()
      ↓
    子进程继承了父进程的全部内存映射（Copy-on-Write）
      ├── ART 虚拟机实例（已初始化好，不需要重新创建）
      ├── 15000 个预加载类（共享只读内存页，不额外占用 RAM）
      └── 共享资源
      ↓
    子进程只需加载应用自己的 dex 和资源 → 启动极快
```

> 如果像标准 JVM 那样每个应用独立启动一个虚拟机实例、独立加载所有类库，在手机上光启动就要好几秒，且每个应用独占一份类库内存，32 个应用就是 32 份——内存直接爆炸。

## 三、Dex 字节码 vs Java 字节码

### 3.1 指令架构差异

**Java 字节码：基于栈（Stack-based）**

所有操作数通过操作数栈（Operand Stack）传递，指令本身不带操作数地址：

```java
// 源码: int c = a + b;
```

```
// Java 字节码:
iload_1        // 把局部变量表 slot1（a）压入操作数栈
iload_2        // 把局部变量表 slot2（b）压入操作数栈
iadd           // 弹出栈顶两个值相加，结果压回栈
istore_3       // 弹出栈顶结果，存入局部变量表 slot3（c）
// 4 条指令，每条 1 字节，共 4 字节
```

**Dex 字节码：基于寄存器（Register-based）**

操作数直接通过虚拟寄存器编号指定，内嵌在指令中：

```
// Dex 字节码:
add-int v0, v1, v2    // v0 = v1 + v2
// 1 条指令，2 字节（opcode 1B + 寄存器编码 1B）
```

**性能对比：**

| 维度 | 栈模型（JVM） | 寄存器模型（ART） |
|------|-------------|-----------------|
| 指令数量 | 多（每个操作都要 push/pop） | 少（约减少 47%） |
| 单条指令长度 | 短（1-3 字节） | 长（2-6 字节） |
| 总代码体积 | 略小 | 略大（约多 25%） |
| 解释执行速度 | 较慢（指令分派开销大） | 较快（指令少，分派次数少） |
| 实现复杂度 | 简单（栈操作直观） | 较复杂（需要寄存器分配） |

> 关键洞察：虽然寄存器模型的代码体积更大，但**执行时的瓶颈是指令分派（dispatch）次数而非代码体积**。每次取指→解码→执行的循环开销是固定的，指令数减少 47% 意味着这个循环少走 47%，因此解释执行模式下寄存器模型显著更快。

### 3.2 文件格式差异

**Java .class 文件**：一个类一个文件，各自维护常量池。

**Dex .dex 文件**：多个类合并为一个文件，全局共享各类 ID 池。

```
.dex 文件结构:
┌──────────────────────────────┐
│ header                       │  文件头（magic/checksum/大小等）
├──────────────────────────────┤
│ string_ids[]                 │  所有字符串的全局去重池
├──────────────────────────────┤
│ type_ids[]                   │  所有类型引用（指向 string_ids）
├──────────────────────────────┤
│ proto_ids[]                  │  方法原型（参数类型 + 返回类型）
├──────────────────────────────┤
│ field_ids[]                  │  字段引用
├──────────────────────────────┤
│ method_ids[]                 │  方法引用（上限 65536，这就是 64K 方法数限制的来源）
├──────────────────────────────┤
│ class_defs[]                 │  类定义（指向上面各池的索引）
├──────────────────────────────┤
│ data                         │  实际的字节码、注解、调试信息等
└──────────────────────────────┘
```

> **64K 方法数限制**：`method_ids` 使用 16 位索引（0~65535），单个 .dex 文件最多引用 65536 个方法。超出后需要 MultiDex（多个 classes.dex、classes2.dex...）。Android 5.0+ 的 ART 原生支持 MultiDex。

### 3.3 方法调用指令对比

```java
// 源码: String result = obj.toString();
```

```
// Java 字节码（栈模型）:
aload_1                       // 把 obj 压栈
invokevirtual #2              // 从栈顶取 this，查常量池 #2 找方法，调用
                              // 返回值压栈
astore_2                      // 弹出结果存入局部变量
// 3 条指令

// Dex 字节码（寄存器模型）:
invoke-virtual {v1}, Ljava/lang/Object;->toString()Ljava/lang/String;
move-result-object v0
// 2 条指令，方法描述符直接内嵌
```

## 四、编译策略的三代演进

这是 ART 最核心的设计演进，直接影响应用的安装速度、启动速度、运行性能和存储占用。

### 4.1 第一代：Dalvik — 纯解释 + JIT（Android 2.2~4.4）

```
安装时:  仅做 dexopt（字节码校验 + 对齐优化）→ 生成 odex
运行时:  解释执行字节码
         ↓ 热点检测（方法调用计数器）
         对热点方法做 JIT 编译为机器码
         ↓
         编译结果仅存在内存中，进程退出即丢失
```

| 优点 | 缺点 |
|------|------|
| 安装快（几乎不做编译） | 运行时编译占 CPU 和电量 |
| 占用存储少 | 每次启动都要重新 JIT 热点代码 |
| | GC 会 Stop-The-World，界面卡顿明显 |

### 4.2 第二代：ART 全量 AOT（Android 5.0~6.0）

Android 5.0 用 ART 完全替代 Dalvik，策略大转向：

```
安装时:  dex2oat 将整个 .dex 文件 AOT 编译为机器码（.oat 文件）
         ↓ 时间代价：安装一个应用可能需要数分钟
运行时:  直接执行预编译的机器码
         ↓ 无需解释执行，无需 JIT
         运行速度极快
```

| 优点 | 缺点 |
|------|------|
| 运行时性能极佳（纯机器码） | 安装极慢（全量编译） |
| 不消耗运行时 CPU 做编译 | 存储占用大（.oat 是 .dex 的 5-8 倍） |
| | 系统 OTA 升级后需要重新编译所有应用（"正在优化应用"） |
| | 冷代码也被编译，浪费存储和安装时间 |

> Android 5.0 时代"正在优化应用 1/120..."的等待画面，就是因为系统升级后要对所有应用执行全量 dex2oat。

### 4.3 第三代：混合编译 — JIT + Profile-Guided AOT（Android 7.0+）

这是目前 ART 使用的策略，结合了前两代的优点：

```
安装时:  不做 AOT 编译，只提取 .dex → 快速安装

首次运行:
  ① 解释执行（全部字节码）
  ② JIT 编译器检测热点方法，编译为机器码缓存在内存中
  ③ 同时将热点方法信息记录到 Profile 文件（.prof）

设备空闲 + 充电时:
  ④ 后台 dex2oat 服务读取 Profile
  ⑤ 仅对 Profile 中的热点方法做 AOT 编译 → .oat 文件
  ⑥ 这就是 Profile-Guided Compilation

后续运行:
  ├── 热点方法 → 直接执行 AOT 机器码（最快）
  ├── 温热方法 → JIT 编译后执行
  └── 冷方法   → 解释执行（可能永远不会被编译）
```

| 维度 | Dalvik JIT | ART 全量 AOT | ART 混合编译 |
|------|-----------|-------------|-------------|
| 安装速度 | 快 | 极慢 | 快 |
| 首次启动 | 慢（纯解释） | 快（已编译） | 中等（解释+JIT） |
| 后续启动 | 慢（每次重新JIT） | 快 | 快（热点已AOT） |
| 存储占用 | 小 | 大（5-8x） | 中（仅编译热点） |
| 运行时功耗 | 高（JIT 耗 CPU） | 低 | 低（空闲时编译） |
| 系统升级 | 快 | 极慢（重编译所有应用） | 快（渐进式重建 Profile） |

### 4.4 Android 12+ 的进一步优化

**Cloud Profiles**：Google Play 收集同一应用在大量设备上的热点 Profile，合并为"云端 Profile"。新用户安装应用时，直接下载云端 Profile，首次启动就能用 AOT 编译的热点代码，不需要经历"冷启动 → 收集 Profile"的阶段。

**Baseline Profile**：开发者可以在应用中主动声明"哪些代码路径是关键的"，打包进 APK：

```
// baseline-prof.txt（由 Macrobenchmark 生成或手动编写）
HSPLcom/example/MyApp;->onCreate(Landroid/os/Bundle;)V
HSPLcom/example/MainActivity;->onResume()V
PLcom/example/adapter/ItemAdapter;->onBindViewHolder(...)V
```

安装时 ART 会优先编译这些路径，首次启动性能可提升 30-40%。

## 五、dex2oat 编译管线

dex2oat 是 ART 的核心编译器，将 .dex 字节码编译为设备 CPU 架构的原生机器码。

### 5.1 编译流程

```
输入: .dex 文件 (+ Profile 文件，如果有的话)
  │
  ├── ① 加载与校验
  │     验证 dex 文件完整性、字节码合法性
  │
  ├── ② 类解析与初始化
  │     解析类层次结构、字段布局、vtable
  │
  ├── ③ 编译（根据编译过滤器选择范围）
  │     ├── Quick Compiler（早期，已淘汰）
  │     └── Optimizing Compiler（当前默认）
  │           ├── 构建 HIR（High-level IR）
  │           ├── 优化 Pass（内联、常量折叠、死代码消除、
  │           │           循环优化、null check 消除等）
  │           ├── 转为 LIR（Low-level IR）
  │           ├── 寄存器分配（Linear Scan）
  │           └── 代码生成（arm64/arm/x86_64/x86）
  │
  ├── ④ 生成 oat 文件
  │     ELF 格式，包含编译后的机器码 + 元数据
  │
  └── ⑤ 生成 art 文件（可选）
        预初始化的堆镜像（类对象、字符串常量池等）
```

### 5.2 编译过滤器（Compiler Filter）

dex2oat 并非总是全量编译，通过过滤器控制编译范围：

| 过滤器 | 含义 | 使用场景 |
|--------|------|---------|
| `verify` | 仅校验字节码，不编译 | 首次安装（Android 7.0+） |
| `quicken` | 校验 + 指令级优化（不生成机器码） | 快速优化 |
| `speed-profile` | 仅编译 Profile 中的热点方法 | 后台空闲时编译（默认） |
| `speed` | 编译所有方法 | 系统应用、`force-compile` |
| `everything` | 编译一切（含调试信息） | 调试/测试 |

```bash
# 手动触发不同级别的编译
adb shell cmd package compile -m speed-profile -f com.example.app
adb shell cmd package compile -m speed -f com.example.app

# 查看应用当前的编译状态
adb shell dumpsys package com.example.app | grep -A 5 "dexopt"
```

## 六、ART 的 GC 机制

### 6.1 与 JVM GC 的核心差异

| 维度 | JVM（HotSpot） | ART |
|------|---------------|-----|
| 分代策略 | 严格分代（Young/Old/Perm→Meta） | 分代但更灵活（Android 10+ 引入分代） |
| GC 算法 | 多种收集器可选（CMS/G1/ZGC） | 针对移动场景优化的专用收集器 |
| 核心约束 | 吞吐量优先或延迟优先 | **延迟优先**（不能让 UI 卡顿） |
| 堆大小 | 通常 GB 级 | 通常 256MB~512MB |
| 内存压力 | 相对宽裕 | 极度敏感（LMK 随时杀进程） |

### 6.2 ART GC 收集器演进

**Dalvik 时代：Mark-Sweep**

```
GC 过程:
  ① Stop-The-World（暂停所有线程）
  ② Mark：从 GC Roots 遍历标记存活对象
  ③ Sweep：回收未标记的对象
  ④ 恢复所有线程

问题：STW 暂停时间长（10~50ms），导致明显卡顿（丢帧）
```

**ART（Android 5.0~7.0）：CMS（Concurrent Mark Sweep）**

```
  ① STW 暂停（短暂）：标记 GC Roots 直接引用
  ② 并发标记：与应用线程并行遍历对象图
  ③ STW 暂停（短暂）：重新标记并发期间变化的引用
  ④ 并发清除：与应用线程并行回收

优化：大部分工作与应用线程并发执行，STW 缩短到 2~5ms
```

**ART（Android 8.0+）：Concurrent Copying（CC）**

```
  核心思想：在 GC 期间将存活对象从 From 区复制到 To 区，
           同时应用线程继续运行

  关键技术 — Read Barrier（读屏障）：
    每次应用线程读取对象引用时，检查该对象是否已被移动
    如果已移动 → 自动转发到新地址（Brooks Pointer）

  优势：
    - 几乎零停顿（STW < 1ms，通常 < 100μs）
    - GC 后内存连续（无碎片），有利于分配性能
    - 特别适合大堆和频繁分配的场景
```

> CC 收集器是 Android 8.0 引入的革命性改进。它使得 GC 引起的卡顿几乎不可感知，是 Android 流畅性提升的重要因素之一。

### 6.3 堆空间布局

```
ART 堆空间:
┌────────────────────────────────────────────────┐
│ Image Space（只读）                              │
│   boot.art 预加载的类对象和字符串                   │
├────────────────────────────────────────────────┤
│ Zygote Space（共享只读，fork 后 COW）              │
│   Zygote 进程中分配的对象                          │
├────────────────────────────────────────────────┤
│ Allocation Space / Region Space                │
│   应用运行时分配的对象（GC 主要管理区域）             │
│   ├── TLAB（Thread Local Allocation Buffer）    │
│   │    每个线程独占的分配缓冲区，无需加锁             │
│   └── 共享区域（TLAB 用完后 fallback）             │
├────────────────────────────────────────────────┤
│ Large Object Space                             │
│   大对象（>12KB）直接分配在此，避免移动开销           │
└────────────────────────────────────────────────┘
```

## 七、运行时文件体系

ART 编译产出多种文件，理解它们的关系对分析安装、启动、OTA 问题很重要：

| 文件 | 格式 | 内容 | 生成时机 |
|------|------|------|---------|
| `.dex` | Dex 字节码 | 应用的原始字节码 | APK 打包时（D8/R8） |
| `.vdex` | 提取的 dex | 从 APK 中提取的原始/校验后 dex | 安装时 |
| `.odex`（新含义） | OAT/ELF | 编译后的机器码 | dex2oat 编译时 |
| `.art` | 堆镜像 | 预初始化的类对象、intern 字符串 | dex2oat 编译时 |
| `.prof` | Profile | 热点方法/类信息 | 运行时 JIT 收集 |
| `.dm` | Dex Metadata | Play 下发的云端 Profile | 应用安装时（来自 Play） |

```
文件关系:
  APK 中的 classes.dex
    │
    ├── 安装时提取 → .vdex（原始 dex 备份，避免反复从 APK 解压）
    │
    ├── JIT 运行时收集 → .prof（热点 Profile）
    │
    └── dex2oat 编译（读取 .vdex + .prof）
          ├── → .odex（编译后的机器码，ELF 格式）
          └── → .art（预初始化的堆快照）

运行时加载顺序:
  ① 加载 .art（直接映射预初始化堆到内存，免去类初始化）
  ② 加载 .odex（映射机器码到内存）
  ③ 对于未编译的方法，回退到 .vdex 中的 dex 字节码解释执行
```

> `.vdex` 的设计目的是避免每次执行时都从 APK（本质是 zip）中解压 dex。直接 mmap .vdex 文件即可访问字节码，大幅减少 I/O。

## 八、运行时优化技术

### 8.1 内联缓存（Inline Cache）

ART 在 JIT 编译和 AOT 编译时都会利用内联缓存优化虚方法调用：

```java
// 虚方法调用（通常需要查 vtable）
animal.speak();  // animal 可能是 Dog、Cat、Bird...
```

```
未优化:  每次调用都查 vtable → 间接跳转 → 慢

单态内联缓存（Monomorphic）:
  if (animal.class == Dog) {
    Dog.speak()     // 直接调用，无需查 vtable
  } else {
    vtable lookup   // fallback
  }

多态内联缓存（Polymorphic）:
  if (animal.class == Dog)    → Dog.speak()
  elif (animal.class == Cat)  → Cat.speak()
  else                        → vtable lookup
```

Profile 文件中会记录每个调用点实际遇到的类型，dex2oat 据此生成优化后的代码。

### 8.2 方法内联

ART 的 Optimizing Compiler 会将小方法直接内联到调用者中：

```java
// 编译前:
public int getSize() { return this.size; }
public void process() {
    int s = getSize();  // 虚方法调用
}

// 内联后（ART 编译器自动完成）:
public void process() {
    int s = this.size;  // 消除了方法调用开销
}
```

内联的判断依据：方法体大小、调用频次（来自 Profile）、是否可去虚化（devirtualize）。

### 8.3 其他优化

- **Null Check Elimination**：如果已知引用非空（如刚 new 出来的对象），跳过空指针检查
- **Bounds Check Elimination**：循环中数组边界检查外提（只检查一次）
- **常量折叠与传播**：编译期计算常量表达式
- **死代码消除**：移除不可达代码
- **循环优化**：循环展开、循环向量化（ARM NEON）

## 九、常见面试题与解答

### Q1：Dalvik 和 ART 有什么区别？

**A：** 核心区别在编译策略和 GC。Dalvik 运行时使用 JIT 编译——每次启动都从解释执行开始，检测到热点方法后 JIT 编译为机器码，但编译结果不持久化，进程退出即丢失。ART 在 Android 5.0~6.0 采用全量 AOT——安装时用 dex2oat 把整个 dex 编译为机器码，运行时直接执行；缺点是安装慢、占存储。Android 7.0+ ART 改为混合模式：安装时不编译，运行时 JIT + 收集 Profile，设备空闲时后台 AOT 编译 Profile 中的热点方法。GC 方面，Dalvik 的 Mark-Sweep 需要长时间 STW（10~50ms），ART 引入了 CMS 和后来的 Concurrent Copying（CC），STW 缩短到 < 1ms。

### Q2：为什么 Android 使用寄存器架构而非栈架构？

**A：** 性能和效率考量。寄存器模型的指令数比栈模型减少约 47%——栈模型每个操作都需要 push/pop 操作数栈，寄存器模型直接在指令中指定寄存器编号。虽然寄存器指令单条更长（代码体积约大 25%），但执行时的瓶颈是指令分派次数而非代码体积。在解释执行模式下（每条指令都有 fetch-decode-execute 的循环开销），指令数减少直接意味着执行更快。此外，手机 CPU 本身就是寄存器架构（ARM），寄存器模型的字节码更容易高效映射到物理寄存器。

### Q3：64K 方法数限制是怎么回事？MultiDex 的原理？

**A：** .dex 文件中 `method_ids` 表使用 16 位索引（范围 0~65535），因此单个 dex 文件最多引用 65536 个方法。当应用引入大量第三方库后很容易超出。MultiDex 方案：D8/R8 在编译时将方法分散到多个 dex 文件（classes.dex、classes2.dex...）。Android 5.0+ 的 ART 原生支持加载多个 dex（dex2oat 一次性编译所有 dex）。Android 4.x 的 Dalvik 需要 `MultiDex.install()` 在运行时通过反射修改 ClassLoader 的 `dexElements` 数组，将额外的 dex 文件追加到类查找路径中。

### Q4：Profile-Guided Optimization (PGO) 在 ART 中是怎么工作的？

**A：** ART 的 PGO 分三步。第一步：运行时 JIT 编译器在编译热点方法时，同时将热点信息（方法引用、类的类型信息、内联缓存数据）写入 Profile 文件（.prof）。第二步：当设备空闲且充电时，系统触发后台 `bg-dex2oat` 任务，读取 Profile 文件，仅对 Profile 中标记的方法执行 AOT 编译（`speed-profile` 过滤器）。第三步：下次启动时，热点方法直接执行 AOT 机器码，冷方法仍然解释执行。此外，Google Play 会聚合大量设备的 Profile 生成 Cloud Profile，新用户安装时直接使用，跳过冷启动收集阶段。开发者也可以通过 Baseline Profile 主动声明关键路径。

### Q5：ART 的 Concurrent Copying GC 为什么能做到几乎零停顿？

**A：** CC GC 的核心技术是 Read Barrier（读屏障）。传统 CMS 在并发标记阶段需要 Write Barrier 跟踪引用变化，但仍需要 STW 来处理 re-mark。CC GC 在对象被移动（从 From 区复制到 To 区）时，通过每个对象头部的 Brooks Pointer（转发指针）记录新地址。应用线程每次读取对象引用时，Read Barrier 自动检查该对象是否已被移动，如果是则透明转发到新地址。这样 GC 线程和应用线程可以真正并发运行，STW 仅用于极短暂的根集扫描（< 100μs）。代价是每次对象读取多了一次指针检查，但 ARM 架构下这个开销很小。

### Q6：.vdex、.odex、.art 文件分别是什么？

**A：** .vdex 是从 APK 中提取的原始 dex 字节码（避免每次都从 zip 解压），通过 mmap 映射到内存供解释器使用。.odex 是 dex2oat 编译产出的 ELF 格式文件，包含编译后的本地机器码。.art 是堆快照镜像文件，包含预初始化的类对象和 intern 字符串，加载时直接映射到内存，免去类加载和初始化的开销。运行时的加载顺序：先加载 .art（恢复预初始化堆），再加载 .odex（映射机器码），未编译的方法回退到 .vdex 解释执行。

### Q7：什么是 Baseline Profile？它与 Cloud Profile 有什么区别？

**A：** Baseline Profile 是开发者在 APK 中主动打包的一份热点方法声明（通常由 Macrobenchmark 库自动生成），告诉 ART "这些代码路径是关键的、需要优先编译"。安装时 ART 读取 Baseline Profile 立即执行 AOT 编译。Cloud Profile 是 Google Play 收集同一应用在大量设备上的运行时 Profile 后合并生成的，随 APK 元数据（.dm 文件）一起下发。两者区别：Baseline Profile 由开发者控制、确定性强、适合关键启动路径；Cloud Profile 基于真实用户行为、覆盖面更广、但有滞后性（需要足够多用户产生数据）。两者可以叠加使用。
