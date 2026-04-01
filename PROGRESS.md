# 知识库建设进度

> 本文件记录知识库的建设状态，供 AI 助手了解当前进展，避免重复劳动。
> 每次新增或大幅修改文档后，请同步更新此文件。

## 已完成的文档

| 文档 | 路径 | 覆盖内容 | 完成度 |
|------|------|---------|--------|
| App 冷启动流程 | `Android/Framework/app启动流程.md` | 五阶段全流程、启动度量、优化实战（含 AsyncLayoutInflater 原理、View 预加载+MutableContextWrapper）、10 道面试题 | 高 |
| 消息机制 | `Android/Framework/消息机制.md` | Handler/Looper/MessageQueue 源码分析、Message 对象池、同步屏障与 VSync、IdleHandler、epoll 底层、与协程关系、9 道面试题 | 高 |
| 类加载机制 | `Android/Java与Kotlin基础/类加载机制.md` | JVM 类加载、双亲委派、打破双亲委派、Android ClassLoader 体系、dexElements 原理、7 道面试题 | 高 |
| 并发编程与线程安全 | `Android/Java与Kotlin基础/并发编程与线程安全.md` | JMM 三大特性、volatile 语义、synchronized 锁升级、AQS 原理、CAS/原子类、ThreadLocal、线程池调优、Kotlin 协程、7 道面试题 | 高 |
| Android 页面绘制 | `Android/UI与渲染/Android页面绘制.md` | View 三大流程（measure/layout/draw）、三进程架构类关系、VSync 渲染流水线、BufferQueue 机制、Fence 同步、Buffer Holding、8 道面试题 | 高 |
| Binder 机制 | `Android/Framework/Binder机制.md` | 设计动机与 Linux IPC 对比、四层架构、mmap 一次拷贝原理（binder_mmap/binder_transaction 源码）、通信全流程调用链、AIDL Stub/Proxy 生成代码分析、ServiceManager bootstrap 机制、Binder 线程池、TransactionTooLargeException 深度分析、DeathRecipient 死亡通知、IPC 方式对比、11 道面试题 | 高 |
| 事件分发机制 | `Android/UI与渲染/事件分发机制.md` | 输入系统全链路（硬件→InputDispatcher→ViewRootImpl）、Activity/ViewGroup/View 三层分发源码、mFirstTouchTarget 链表、onInterceptTouchEvent 调用时机、滑动冲突解决（外部拦截/内部拦截/NestedScrolling）、多点触摸分发、9 道面试题 | 高 |
| RecyclerView 缓存机制 | `Android/UI与渲染/RecyclerView缓存机制.md` | 架构哲学与组件解耦、四级缓存机制源码分析、缓存查找与回收流程、LayoutManager 与 Recycler 交互（fill/scrollBy）、ItemDecoration/ItemAnimator/SnapHelper、DiffUtil Myers 算法与 AsyncListDiffer 线程模型、性能优化实战（stableIds/payloads/共享 Pool/Prefetch 调优）、RV vs LV 对比、10 道面试题 | 高 |
| Kotlin 协程原理 | `Android/Java与Kotlin基础/Kotlin协程原理.md` | CPS 变换与状态机反编译、CoroutineScope/launch/async 启动流程、CoroutineScheduler 工作窃取与 IO/Default 共享线程池、Dispatchers.Main Handler 调度、Job 状态机与取消传播、SupervisorJob、异常处理两路径与 CEH 安装规则、Mutex/Channel/select、Android Scope 选型与 repeatOnLifecycle、10 道面试题 | 高 |
| Kotlin Flow 深度解析 | `Android/Java与Kotlin基础/Kotlin Flow深度解析.md` | 冷流原理（SafeFlow/上下文保持/channelFlow/callbackFlow）、操作符套娃结构与 flatMap 系列、背压策略（buffer/conflate/collectLatest）、flowOn Channel 桥接原理、StateFlow/SharedFlow 内部实现与配置参数、异常处理（catch/retry/onCompletion）、stateIn 策略、Android 架构模式、Flow vs LiveData vs RxJava、10 道面试题 | 高 |
| Activity 生命周期与启动模式 | `Android/Framework/Activity生命周期与启动模式.md` | 七大回调与状态机、ClientTransaction 驱动机制源码、TransactionExecutor 状态推进、onSaveInstanceState 时机变化、边界场景（跳转/Dialog/透明/Fragment/多窗口/配置变更/进程死亡）、五种启动模式对比与源码分析、Task 与 taskAffinity、Intent Flags 组合、10 道面试题 | 高 |
| 内存泄漏与内存优化 | `Android/性能优化/内存泄漏与内存优化.md` | 进程内存模型、分代回收与 GC 类型、LMK 机制、7 种泄漏场景（Handler/匿名内部类/静态引用/监听器/WebView/集合）、LeakCanary 原理（WeakReference+ReferenceQueue+Shark）、Profiler/MAT/dumpsys、Bitmap 优化（inSampleSize/inBitmap/Hardware）、内存抖动与对象池、LruCache、Native 内存管理、OOM 分类与治理、KOOM 线上监控、10 道面试题 | 高 |
| WMS 与 PMS 核心原理 | `Android/Framework/WMS与PMS核心原理.md` | WMS 架构与核心类、窗口类型与层级树、窗口添加流程源码、WindowToken 机制、窗口动画、PMS 开机扫描、APK 安装流程与签名验证(v1-v4)、dex2oat 编译、权限模型演进、Intent 解析机制、10 道面试题 | 高 |
| 技能图谱 | `Android/技能图谱.md` | 十一大章节的完整大纲（含端侧AI与大模型），5/7/10 年能力分层参考 | 完整（持续更新索引） |
| ViewModel 数据保持原理 | `Android/Jetpack与架构/ViewModel数据保持原理.md` | NonConfigurationInstances 存活机制、ViewModelStore/ViewModelProvider 源码、Factory 体系（CreationExtras）、SavedStateHandle 与 SavedStateRegistry、onCleared 时机、viewModelScope、Fragment 共享、8 道面试题 | 高 |
| OkHttp 拦截器链原理 | `Android/开源框架原理/OkHttp拦截器链原理.md` | 责任链模式、RealInterceptorChain 源码、5 大内置拦截器详解、连接池复用、Application vs Network Interceptor、实战示例（Token/日志/刷新）、8 道面试题 | 高 |
| Glide 缓存机制 | `Android/开源框架原理/Glide缓存机制.md` | 三级缓存（ActiveResources/MemoryCache/DiskCache）、**ActiveResources 引用计数保护机制**、生命周期感知（无UIFragment）、BitmapPool 复用、**缓存 Key 双体系（内存 EngineKey 精确匹配/磁盘 Data Key 原图匹配+Resource Key 转换匹配）**、**五种 DiskCacheStrategy 与作用层级（仅磁盘层，内存层独立控制）**、与 Coil 对比、7 道面试题 | 高 |
| JVM 内存模型 | `Android/Java与Kotlin基础/JVM内存模型.md` | 运行时数据区（PC/栈/堆/方法区）、栈帧结构、堆内存划分与晋升、方法区演进（永久代→元空间）、直接内存、OOM 场景分析、与 Android ART 对比、7 道面试题 | 高 |
| 垃圾回收机制 | `Android/Java与Kotlin基础/垃圾回收机制.md` | GC Roots 可达性分析、四种引用类型、三种收集算法、CMS/G1/ZGC 收集器对比、GC 日志分析、与 Android ART 对比、7 道面试题 | 高 |
| HashMap 深度解析 | `Android/Java与Kotlin基础/HashMap深度解析.md` | 底层结构（数组+链表+红黑树）、put/get 流程、扩容机制、红黑树转换条件、线程安全问题、LinkedHashMap/TreeMap 变体、7 道面试题 | 高 |
| 泛型深度解析 | `Android/Java与Kotlin基础/泛型深度解析.md` | 类型擦除机制、桥接方法、三种通配符、PECS 原则、泛型限制、TypeToken 模式、与 Kotlin 泛型对比、7 道面试题 | 高 |
| 反射与注解深度解析 | `Android/Java与Kotlin基础/反射与注解深度解析.md` | 反射核心 API、动态代理（JDK/CGLIB）、注解机制、元注解、APT 编译时处理、KSP、7 道面试题 | 高 |
| Kotlin 语言特性深度解析 | `Android/Java与Kotlin基础/Kotlin语言特性深度解析.md` | 空安全（类型系统/平台类型/智能转换/Contracts）、扩展函数（字节码/静态分发/作用域函数5大对比）、密封类/数据类/值类/枚举对比、委托（by lazy DCL/observable/provideDelegate/自定义SP委托）、内联函数（inline/noinline/crossinline/reified）、DSL构建（Lambda with receiver/@DslMarker）、泛型型变（out/in/星投影/Java PECS对比）、10 道面试题 | 高 |
| Service 深度解析 | `Android/Framework/Service深度解析.md` | startService/bindService 生命周期差异及源码调用链、onStartCommand 返回值策略、Bound Service 三种实现（Binder/Messenger/AIDL）、前台服务类型（Android 14+）、后台限制演进时间线（Android 6~15）、IntentService 废弃与替代方案对比、进程优先级与保活争议、WorkManager 对比、7 道面试题 | 高 |
| BroadcastReceiver 深度解析 | `Android/Framework/BroadcastReceiver深度解析.md` | 静态/动态注册源码调用链、广播分发流程（AMS broadcastIntentLocked）、有序广播拦截机制、goAsync() 原理、Android 8.0+ 隐式广播限制及应对、各版本限制汇总、LocalBroadcastManager 废弃与 SharedFlow 替代、Android 13+ exported 标志、6 道面试题 | 高 |
| ContentProvider 深度解析 | `Android/Framework/ContentProvider深度解析.md` | 极早的初始化时机（早于 Application.onCreate）、跨进程通信全流程（ContentResolver→AMS→Provider）、CursorWindow 共享内存传递、unstable/stable Provider、URI 路由与 UriMatcher、权限模型与临时 URI 授权、FileProvider、App Startup 合并初始化原理、ContentObserver 变更通知、6 道面试题 | 高 |
| ANR 原理与分析 | `Android/性能优化/ANR原理与分析.md` | 四大 ANR 类型与超时阈值、Input ANR（InputDispatcher Native 层）/Broadcast/Service/ContentProvider 触发机制源码分析、埋弹-拆弹模型、ANR 处理流程（SIGQUIT/traces.txt/DropBox）、traces.txt 结构解析与堆栈模式、三步定位法（CPU→堆栈→上下文）、线上监控方案对比（Looper Printer/ANR-WatchDog/SIGQUIT 监听）、预防最佳实践、10 道面试题 | 高 |
| 架构演进 MVC-MVP-MVVM-MVI | `Android/Jetpack与架构/架构演进MVC-MVP-MVVM-MVI.md` | MVC 在 Android 中的退化（V+C 二合一）、MVP 接口解耦与痛点（接口爆炸/生命周期/内存泄漏）、MVVM 观察者模式与生命周期感知、MVVM 状态分散问题、MVI 单向数据流与不可变状态（State/Intent/Effect）、State vs Effect 划分原则、Reducer 模式、State 膨胀应对策略、四代架构横向对比表、Google UDF 架构指南、渐进式迁移策略、实战选型建议、10 道面试题 | 高 |
| Gradle 构建流程与 APK 编译 | `Android/工程化与DevOps/Gradle构建流程与APK编译.md` | 三阶段生命周期（初始化/配置/执行）、Task DAG 调度模型、APK 编译管线全流程（AAPT2→kotlinc/javac→D8/R8→打包→签名）、AAPT2 增量编译、D8 vs R8 对比、脱糖原理、Build Variant、Groovy vs Kotlin DSL、Version Catalog、Convention Plugin vs buildSrc 选型、自定义 Plugin/Task 与增量构建、AsmClassVisitorFactory 字节码插桩、KSP vs kapt、构建加速全景（Configuration Cache/Build Cache/并行构建）、依赖管理（implementation vs api）、**依赖冲突解决机制（Highest Version Wins/force/exclude/substitution）、SDK 依赖管理策略（compileOnly/BOM/Shadow Relocate 字节码重写原理）、Jetifier 原理与迁移、依赖排查命令**、11 道面试题 | 高 |
| ART 虚拟机原理 | `Android/底层原理与系统源码/ART虚拟机原理.md` | 为什么不用标准JVM、Zygote fork 共享模型、Dex vs Java 字节码（栈vs寄存器架构/文件格式/指令对比）、编译策略三代演进（Dalvik JIT→ART全量AOT→混合编译PGO）、Cloud Profile/Baseline Profile、dex2oat编译管线与过滤器、ART GC演进（Mark-Sweep→CMS→Concurrent Copying/Read Barrier）、堆空间布局（Image/Zygote/Allocation/LargeObject）、运行时文件体系（vdex/odex/art/prof/dm）、运行时优化（内联缓存/方法内联/NullCheck消除）、7 道面试题 | 高 |
| ASM 字节码插桩 | `Android/工程化与DevOps/ASM字节码插桩.md` | AOP与插桩概念、ASM在Android工具链中的位置、**.class文件结构与类型描述符**、**JVM栈帧模型与操作数栈执行过程**、**常用字节码指令速查（加载/存储/算术/对象/栈操作/控制流）**、ASM两套API（Core vs Tree）对比、**Core API访问者模式流水线（ClassReader→ClassVisitor→MethodVisitor→ClassWriter）**、AdviceAdapter原理、**AsmClassVisitorFactory接口详解与参数传递**、InstrumentationScope/FramesComputationMode配置、Composite Build工程结构、**完整实战：@LogTime注解方法耗时统计（TimingMethodVisitor逐条指令注释栈状态）**、常见插桩场景（隐私合规/点击防抖/线程检查/全埋点）、8道面试题 | 高 |


## 历史遗留文档（已迁移为 Markdown）

以下 HTML/PDF 文档的内容已全部迁移到对应的 Markdown 文档中，可安全删除：

| 旧文件 | 迁移目标 |
|--------|---------|
| `Android/UI与渲染/Android页面绘制_美化版.html` | `Android/UI与渲染/Android页面绘制.md` |
| `Android/UI与渲染/Android页面绘制.html` | 同上（早期版本） |
| `Android/UI与渲染/Android页面绘制_美化版.pdf` | 同上（PDF 导出版） |
| `Android/Java与Kotlin基础/多线程&线程安全.html` | `Android/Java与Kotlin基础/并发编程与线程安全.md` |
| `Android/Framework/消息队列.html` | `Android/Framework/消息机制.md` |

## 待建设的重点领域

按技能图谱优先级排序：

### 高优先级（大纲核心章节，尚无文档）

- [x] Binder 机制（IPC 核心，面试极高频）
- [x] 事件分发机制（UI 核心，面试极高频）
- [x] RecyclerView 缓存机制（面试高频）
- [x] Kotlin 协程原理（挂起函数、调度器、结构化并发）
- [x] Activity 生命周期与启动模式
- [x] 内存泄漏与内存优化
- [x] 核心系统服务：WMS 与 PMS 核心原理

### 中优先级

- [x] ViewModel 数据保持原理
- [x] LiveData / Flow 对比
- [x] OkHttp 拦截器链原理
- [x] Glide 缓存机制
- [x] MVC → MVP → MVVM → MVI 架构演进
- [x] Gradle 构建流程
- [x] ANR 原理与分析
- [ ] 动态化 UI：Server-Driven UI 服务端下发机制与原生架构
- [ ] 研发规范：基于 AST 的自定义质量防线与 Lint 检查

### 低优先级（可后续补充）

- [ ] Jetpack Compose 原理
- [ ] 跨平台技术与多端融合实践（KMP / Flutter / HarmonyOS NEXT）
- [ ] 签名机制
- [x] ART 虚拟机原理
- [ ] JNI/NDK 基础
- [ ] 端侧 AI 推理框架与大模型实践（TFLite / Gemini Nano 等）
- [ ] 逆向与防御：脱壳加壳机制及 Hook 技术探讨
- [ ] 极限场景分析：深水区内存防爆与复杂 IO 读写策略

## 用户偏好备忘

- 用户是有一定基础的 Android 开发者，正在系统性地深化知识体系
- 喜欢从一个知识点出发，追问底层原理（如：预加载的类 → 类加载机制 → 双亲委派）
- 重视面试场景，每篇文档都希望附带面试题
- 习惯在不同设备/AI助手之间切换工作，因此规则和进度必须持久化在项目中
- 不使用 emoji（技能图谱中 📄 索引标记除外）
- 文档要求使用 Markdown 格式，不使用 HTML
- 复杂流程图使用 SVG 矢量图（Mermaid 源码 → mmdc 渲染），不要用 ASCII 硬画复杂图；简单线性调用链可用 ASCII
- **输出大小控制**：由于服务端流式传输限制，对于内容极长（预计超过 300 行）或需要大幅修改的文档，**必须分段生成或采用增量写入模式（replace_file_content）**，严禁在一次回复中尝试输出过大的代码块/文档，以防 `unexpected EOF` 连接超时错误。
