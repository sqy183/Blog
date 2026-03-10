# 知识库建设进度

> 本文件记录知识库的建设状态，供 AI 助手了解当前进展，避免重复劳动。
> 每次新增或大幅修改文档后，请同步更新此文件。

## 已完成的文档

| 文档 | 路径 | 覆盖内容 | 完成度 |
|------|------|---------|--------|
| App 冷启动流程 | `Android/Framework/app启动流程.md` | 五阶段全流程、启动度量、优化实战（含 AsyncLayoutInflater 原理、View 预加载+MutableContextWrapper）、10 道面试题 | 高 |
| 类加载机制 | `Android/Java与Kotlin基础/类加载机制.md` | JVM 类加载、双亲委派、打破双亲委派、Android ClassLoader 体系、dexElements 原理、7 道面试题 | 高 |
| 技能图谱 | `Android/技能图谱.md` | 十一大章节的完整大纲（含端侧AI与大模型），5/7/10 年能力分层参考 | 完整（持续更新索引） |

## 历史遗留文档（非 Markdown 格式）

以下为早期创建的 HTML/PDF 文档，内容有效但格式不统一，后续可考虑迁移为 Markdown：

| 文档 | 路径 |
|------|------|
| Android 页面绘制 | `Android/UI与渲染/Android页面绘制_美化版.html` |
| 多线程与线程安全 | `Android/Java与Kotlin基础/多线程&线程安全.html` |
| 消息队列 | `Android/Framework/消息队列.html` |

## 待建设的重点领域

按技能图谱优先级排序（用户关注 Android 方向）：

### 高优先级（大纲核心章节，尚无文档）

- [ ] Binder 机制（IPC 核心，面试极高频）
- [ ] 事件分发机制（UI 核心，面试极高频）
- [ ] RecyclerView 缓存机制（面试高频）
- [ ] Kotlin 协程原理（挂起函数、调度器、结构化并发）
- [ ] Handler/Looper 消息机制深度解析（已有 HTML，待 Markdown 化并扩展）
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
