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

按技能图谱优先级排序（用户关注 Android 方向）：

### 高优先级（大纲核心章节，尚无文档）

- [x] Binder 机制（IPC 核心，面试极高频）
- [x] 事件分发机制（UI 核心，面试极高频）
- [ ] RecyclerView 缓存机制（面试高频）
- [ ] Kotlin 协程原理（挂起函数、调度器、结构化并发）
- [ ] Activity 生命周期与启动模式
- [ ] 内存泄漏与内存优化

### 中优先级

- [ ] ViewModel 数据保持原理
- [ ] LiveData / Flow 对比
- [ ] OkHttp 拦截器链原理
- [ ] Glide 缓存机制
- [ ] MVC → MVP → MVVM → MVI 架构演进
- [ ] Gradle 构建流程
- [ ] ANR 原理与分析

### 低优先级（可后续补充）

- [ ] Jetpack Compose 原理
- [ ] 跨平台（KMP / Flutter）
- [ ] 签名机制
- [ ] ART 虚拟机
- [ ] JNI/NDK 基础
- [ ] 端侧 AI 推理框架（TFLite / MNN / ONNX Runtime）
- [ ] 端侧大模型实践（Gemini Nano / 模型优化 / 内存管理）

## 用户偏好备忘

- 用户是有一定基础的 Android 开发者，正在系统性地深化知识体系
- 喜欢从一个知识点出发，追问底层原理（如：预加载的类 → 类加载机制 → 双亲委派）
- 重视面试场景，每篇文档都希望附带面试题
- 习惯在不同设备/AI助手之间切换工作，因此规则和进度必须持久化在项目中
- 不使用 emoji（技能图谱中 📄 索引标记除外）
- 文档要求使用 Markdown 格式，不使用 HTML
- 复杂流程图使用 SVG 矢量图（Mermaid 源码 → mmdc 渲染），不要用 ASCII 硬画复杂图；简单线性调用链可用 ASCII
