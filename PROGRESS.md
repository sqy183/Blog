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
| 技能图谱 | `Android/技能图谱.md` | 十一大章节的完整大纲（含端侧AI与大模型），5/7/10 年能力分层参考 | 完整（持续更新索引） |


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
- [ ] Activity 生命周期与启动模式
- [ ] 内存泄漏与内存优化
- [ ] 核心系统服务：WMS 与 PMS 核心原理

### 中优先级

- [ ] ViewModel 数据保持原理
- [x] LiveData / Flow 对比
- [ ] OkHttp 拦截器链原理
- [ ] Glide 缓存机制
- [ ] MVC → MVP → MVVM → MVI 架构演进
- [ ] Gradle 构建流程
- [ ] ANR 原理与分析
- [ ] 动态化 UI：Server-Driven UI 服务端下发机制与原生架构
- [ ] 研发规范：基于 AST 的自定义质量防线与 Lint 检查

### 低优先级（可后续补充）

- [ ] Jetpack Compose 原理
- [ ] 跨平台技术与多端融合实践（KMP / Flutter / HarmonyOS NEXT）
- [ ] 签名机制
- [ ] ART 虚拟机原理
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
