# ViewModel 数据保持原理

## 一、概述

ViewModel 是 Jetpack 架构组件的核心，解决了一个 Android 开发中最基本的问题：**配置变更（如屏幕旋转）导致 Activity/Fragment 重建时，如何保持 UI 相关数据不丢失**。

传统方案的痛点：

| 方案 | 问题 |
|------|------|
| `onSaveInstanceState` | 只能存 Bundle，有大小限制（~1MB），且仅适合轻量级数据 |
| `static` 变量 | 生命周期不可控，容易内存泄漏 |
| `retain Fragment` | API 复杂，已废弃 |

ViewModel 的核心设计：**将数据的生命周期与 Activity/Fragment 的"逻辑生命周期"绑定**。所谓逻辑生命周期，是指跨越配置变更的生命周期 -- Activity 因旋转而销毁重建时，ViewModel 不会被销毁。

> ViewModel 的生存范围：从 Activity 首次创建到 Activity **真正 finish**（用户主动退出或系统回收）。配置变更导致的销毁重建不算 finish。

---

## 二、ViewModel 的存活机制

### 2.1 核心问题：配置变更时数据存在哪？

Activity 重建时，旧 Activity 实例被销毁，新 Activity 实例被创建。ViewModel 能存活的关键在于：**它被存放在一个不随 Activity 销毁而销毁的容器中**。

这个容器的传递链路：

```
旧 Activity 销毁前
  → ActivityThread.performDestroyActivity()
    → Activity.retainNonConfigurationInstances()
      → 将 ViewModelStore 打包到 NonConfigurationInstances 对象中
        → 暂存在 ActivityClientRecord 中

新 Activity 创建时
  → ActivityThread.performLaunchActivity()
    → Activity.attach()
      → 将 NonConfigurationInstances 传入新 Activity
        → 新 Activity 从中取回 ViewModelStore
          → 从 ViewModelStore 取回所有 ViewModel
```

### 2.2 NonConfigurationInstances 详解

`Activity.NonConfigurationInstances` 是一个内部静态类，专门用于在配置变更时保存不需要序列化的数据：

```java
// android.app.Activity
static final class NonConfigurationInstances {
    Object activity;       // ComponentActivity 用这个字段存 ViewModelStore
    HashMap<String, Object> children;
    FragmentManagerNonConfig fragments;
    ArrayMap<String, LoaderManager> loaders;
}
```

配置变更时的保存流程（源码级）：

```java
// ActivityThread.java
void performDestroyActivity(ActivityClientRecord r, ...) {
    // 1. 在销毁前，调用 retainNonConfigurationInstances
    r.lastNonConfigurationInstances = r.activity.retainNonConfigurationInstances();
    // 2. 此时 ActivityClientRecord 还在内存中（它属于 ActivityThread，不会因重建而销毁）
    // 3. 执行 Activity.onDestroy()
}
```

```java
// ComponentActivity.java
@Override
public final Object onRetainNonConfigurationInstance() {
    Object custom = onRetainCustomNonConfigurationInstance();
    ViewModelStore viewModelStore = mViewModelStore;
    if (viewModelStore == null) {
        // 如果当前没有创建 ViewModelStore，检查上一次传入的
        NonConfigurationInstances nc = (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            viewModelStore = nc.viewModelStore;
        }
    }
    if (viewModelStore == null && custom == null) {
        return null;
    }
    NonConfigurationInstances nci = new NonConfigurationInstances();
    nci.custom = custom;
    nci.viewModelStore = viewModelStore;
    return nci;
}
```

关键点：`ActivityClientRecord` 是 `ActivityThread` 的成员变量，它不会因为 Activity 的销毁而消失。当新 Activity 启动时，`ActivityThread` 会把保存的 `NonConfigurationInstances` 通过 `Activity.attach()` 传给新实例。

---

## 三、ViewModelStore 与 ViewModelProvider

### 3.1 ViewModelStore -- ViewModel 的容器

`ViewModelStore` 本质上就是一个 `HashMap`：

```java
public class ViewModelStore {
    private final HashMap<String, ViewModel> mMap = new HashMap<>();

    final void put(String key, ViewModel viewModel) {
        ViewModel oldViewModel = mMap.put(key, viewModel);
        if (oldViewModel != null) {
            oldViewModel.onCleared();
        }
    }

    final ViewModel get(String key) {
        return mMap.get(key);
    }

    Set<String> keys() {
        return new HashSet<>(mMap.keySet());
    }

    // Activity 真正 finish 时调用
    public final void clear() {
        for (ViewModel vm : mMap.values()) {
            vm.clear();  // 内部调用 onCleared()
        }
        mMap.clear();
    }
}
```

### 3.2 ViewModelStoreOwner 接口

Activity 和 Fragment 都实现了 `ViewModelStoreOwner`：

```java
public interface ViewModelStoreOwner {
    ViewModelStore getViewModelStore();
}
```

`ComponentActivity` 的实现：

```java
// ComponentActivity.java
@Override
public ViewModelStore getViewModelStore() {
    if (getApplication() == null) {
        throw new IllegalStateException("...");
    }
    ensureViewModelStore();
    return mViewModelStore;
}

void ensureViewModelStore() {
    if (mViewModelStore == null) {
        NonConfigurationInstances nc =
            (NonConfigurationInstances) getLastNonConfigurationInstance();
        if (nc != null) {
            // 配置变更后，从上一个 Activity 实例恢复 ViewModelStore
            mViewModelStore = nc.viewModelStore;
        }
        if (mViewModelStore == null) {
            mViewModelStore = new ViewModelStore();
        }
    }
}
```

> 这就是 ViewModel 存活的核心机制：新 Activity 创建时，通过 `getLastNonConfigurationInstance()` 拿回旧的 `ViewModelStore`，而非创建新的。

### 3.3 ViewModelProvider -- 创建/获取 ViewModel 的入口

```kotlin
// 使用方式
val viewModel = ViewModelProvider(this)[MyViewModel::class.java]

// 或使用 KTX 扩展
val viewModel: MyViewModel by viewModels()
```

`ViewModelProvider` 的核心逻辑：

```java
public class ViewModelProvider {
    private final Factory mFactory;
    private final ViewModelStore mViewModelStore;

    public <T extends ViewModel> T get(Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }

    public <T extends ViewModel> T get(String key, Class<T> modelClass) {
        ViewModel viewModel = mViewModelStore.get(key);

        if (modelClass.isInstance(viewModel)) {
            // 已存在（配置变更恢复 或 同一 Activity 内再次获取），直接返回
            return (T) viewModel;
        }

        // 不存在，通过 Factory 创建
        viewModel = mFactory.create(modelClass);
        mViewModelStore.put(key, viewModel);
        return (T) viewModel;
    }
}
```

> key 的默认格式是 `"androidx.lifecycle.ViewModelProvider.DefaultKey:com.example.MyViewModel"`，这确保了不同类型的 ViewModel 在同一个 ViewModelStore 中不会冲突。

---

## 四、Factory 机制 -- ViewModel 的创建策略

### 4.1 默认 Factory

如果 ViewModel 没有构造参数，使用 `NewInstanceFactory`：

```java
public static class NewInstanceFactory implements Factory {
    @Override
    public <T extends ViewModel> T create(Class<T> modelClass) {
        try {
            return modelClass.getDeclaredConstructor().newInstance();
        } catch (Exception e) {
            throw new RuntimeException("Cannot create ViewModel", e);
        }
    }
}
```

### 4.2 AndroidViewModelFactory

需要 `Application` 参数时使用：

```java
public static class AndroidViewModelFactory extends NewInstanceFactory {
    private Application mApplication;

    @Override
    public <T extends ViewModel> T create(Class<T> modelClass) {
        if (AndroidViewModel.class.isAssignableFrom(modelClass)) {
            return modelClass.getConstructor(Application.class)
                             .newInstance(mApplication);
        }
        return super.create(modelClass);
    }
}
```

> `AndroidViewModel` 持有 `Application` 引用（注意不是 Activity 引用，不会泄漏），适合需要 Context 的场景。

### 4.3 自定义 Factory -- 依赖注入的桥梁

当 ViewModel 需要自定义依赖时，必须通过自定义 Factory 注入：

```kotlin
class MyViewModelFactory(
    private val repository: UserRepository
) : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        return MyViewModel(repository) as T
    }
}

// 使用
val vm = ViewModelProvider(this, MyViewModelFactory(repo))[MyViewModel::class.java]
```

在使用 Hilt 时，`@HiltViewModel` + `@Inject constructor` 会自动生成 Factory，底层原理相同。

### 4.4 CreationExtras（Lifecycle 2.5+）

新版本引入了 `CreationExtras`，用 key-value 方式传递创建参数，替代了硬编码的 Factory 构造：

```kotlin
class MyViewModel(
    private val savedStateHandle: SavedStateHandle,
    private val repository: UserRepository
) : ViewModel()

class MyFactory : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>, extras: CreationExtras): T {
        val app = extras[APPLICATION_KEY]!!
        val savedStateHandle = extras.createSavedStateHandle()
        return MyViewModel(savedStateHandle, UserRepository(app)) as T
    }
}
```

---

## 五、SavedStateHandle -- ViewModel 的持久化补充

### 5.1 ViewModel 的局限性

ViewModel 只能在**配置变更**时保持数据。当进程被系统杀死（用户按 Home 后系统回收内存）时，ViewModel 也随之消亡。

| 场景 | ViewModel 存活？ | SavedStateHandle 存活？ |
|------|:---:|:---:|
| 屏幕旋转 | 是 | 是 |
| 切到后台再回来（进程未死） | 是 | 是 |
| 进程被杀死后恢复 | **否** | **是** |
| 用户主动按返回退出 | 否 | 否 |

### 5.2 SavedStateHandle 原理

`SavedStateHandle` 本质上是把数据存入 `onSaveInstanceState()` 的 Bundle 中，从而在进程死亡后能恢复。

```
SavedStateHandle
  ↕ 自动同步
SavedStateRegistry（ComponentActivity 持有）
  ↕ onSaveInstanceState / onCreate(savedInstanceState)
Activity 的 Bundle
  ↕ 系统序列化
磁盘持久化
```

核心类：

```java
public final class SavedStateHandle {
    // 常规存储：key-value Map
    final Map<String, Object> mRegular;
    // LiveData 缓存
    private final Map<String, SavingStateLiveData<?>> mLiveDatas = new HashMap<>();

    // 获取 LiveData（响应式读取）
    public <T> MutableLiveData<T> getLiveData(String key) {
        return getLiveDataInternal(key, false, null);
    }

    // 获取 StateFlow（Flow 方式）
    public <T> StateFlow<T> getStateFlow(String key, T initialValue) {
        // 内部基于 getLiveData 实现
    }

    // 直接读写
    public <T> T get(String key) { return (T) mRegular.get(key); }
    public <T> void set(String key, T value) {
        mRegular.put(key, value);
        // 同步通知 LiveData
    }
}
```

### 5.3 SavedStateRegistry -- 注册与恢复

`ComponentActivity` 在其生命周期中自动管理 `SavedStateRegistry`：

```java
// ComponentActivity.java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    // 从 Bundle 恢复所有已注册的 SavedStateProvider 的数据
    mSavedStateRegistryController.performRestore(savedInstanceState);
}

@Override
protected void onSaveInstanceState(Bundle outState) {
    // 将所有 SavedStateProvider 的数据写入 Bundle
    mSavedStateRegistryController.performSave(outState);
    super.onSaveInstanceState(outState);
}
```

### 5.4 使用场景建议

```kotlin
class SearchViewModel(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    // 搜索关键词：进程死亡后需要恢复
    val query: StateFlow<String> = savedStateHandle.getStateFlow("query", "")

    // 搜索结果列表：ViewModel 层面保持即可，进程死亡后重新请求
    private val _results = MutableStateFlow<List<Item>>(emptyList())
    val results: StateFlow<List<Item>> = _results.asStateFlow()

    fun search(keyword: String) {
        savedStateHandle["query"] = keyword  // 自动持久化
        viewModelScope.launch {
            _results.value = repository.search(keyword)
        }
    }
}
```

> 原则：**用户输入型数据**（搜索词、表单填写、滚动位置）存 SavedStateHandle；**可重新获取的数据**（网络请求结果、数据库查询结果）只存 ViewModel 普通字段。

---

## 六、ViewModel 的生命周期管理

### 6.1 onCleared() -- 清理时机

当 Activity **真正 finish** 时（非配置变更），ViewModel 的 `onCleared()` 被调用：

```java
// ComponentActivity 构造函数中注册 Lifecycle 监听
getLifecycle().addObserver(new LifecycleEventObserver() {
    @Override
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        if (event == Lifecycle.Event.ON_DESTROY) {
            // 关键判断：是配置变更还是真正退出？
            if (!isChangingConfigurations()) {
                getViewModelStore().clear();  // 清理所有 ViewModel
            }
        }
    }
});
```

> `isChangingConfigurations()` 返回 true 表示是配置变更（如旋转），此时不清理 ViewModel。只有真正 finish 时才调用 `clear()` → `onCleared()`。

### 6.2 viewModelScope

`viewModelScope` 是 ViewModel 内置的协程作用域，在 `onCleared()` 时自动取消：

```kotlin
// ViewModel.kt
public val ViewModel.viewModelScope: CoroutineScope
    get() {
        val scope = getTag(JOB_KEY) as? CloseableCoroutineScope
        if (scope != null) return scope
        return setTagIfAbsent(JOB_KEY,
            CloseableCoroutineScope(SupervisorJob() + Dispatchers.Main.immediate))
    }

// ViewModel.clear() 内部
for (val closeable : mBagOfTags.values()) {
    closeWithRuntimeException(closeable)  // 取消 viewModelScope
}
onCleared()
```

---

## 七、Fragment 间共享 ViewModel

### 7.1 Activity 级别共享

多个 Fragment 可以通过宿主 Activity 的 ViewModelStore 共享同一个 ViewModel：

```kotlin
// Fragment A
val sharedViewModel: SharedViewModel by activityViewModels()

// Fragment B
val sharedViewModel: SharedViewModel by activityViewModels()

// 两者拿到的是同一个实例，因为都从 Activity 的 ViewModelStore 获取
```

### 7.2 Navigation Graph 级别共享

在使用 Navigation 组件时，可以在导航图级别共享 ViewModel：

```kotlin
val navController = findNavController()
val graphViewModel: SharedViewModel by navGraphViewModels(R.id.my_nav_graph)
```

这适合只在某个导航流程中共享数据的场景（如多步骤表单），而非整个 Activity 级别。

### 7.3 共享边界总结

| 共享方式 | 作用域 | 适用场景 |
|---------|--------|---------|
| `by viewModels()` | 当前 Fragment | 单个页面的私有数据 |
| `by activityViewModels()` | 宿主 Activity | 同一 Activity 下多 Fragment 通信 |
| `by navGraphViewModels(graphId)` | 导航图 | 多步骤流程中的数据共享 |

---

## 八、常见面试题与解答

### Q1: ViewModel 是怎么在屏幕旋转时保持数据的？

**A**: ViewModel 被存放在 `ViewModelStore`（本质是 HashMap）中。配置变更时，`ActivityThread` 在销毁旧 Activity 前调用 `retainNonConfigurationInstances()` 将 `ViewModelStore` 保存到 `ActivityClientRecord` 中。`ActivityClientRecord` 是 `ActivityThread` 的成员，不受 Activity 销毁影响。新 Activity 创建时，通过 `Activity.attach()` 方法将旧的 `NonConfigurationInstances` 传入，新 Activity 从中恢复 `ViewModelStore`，从而拿到之前的所有 ViewModel 实例。

### Q2: ViewModel 的 onCleared() 什么时候调用？

**A**: 只在 Activity **真正 finish** 时调用，配置变更导致的销毁**不会**调用。`ComponentActivity` 在构造函数中注册了 `LifecycleObserver`，监听 `ON_DESTROY` 事件，并通过 `isChangingConfigurations()` 判断是否是配置变更。只有不是配置变更时，才调用 `ViewModelStore.clear()` → `ViewModel.onCleared()`。

### Q3: ViewModel 能否在进程被杀死后恢复数据？

**A**: 不能。ViewModel 保存在内存中的 `ViewModelStore` 里，进程死亡后内存被回收。如果需要在进程死亡后恢复数据，应使用 `SavedStateHandle`，它底层依赖 `onSaveInstanceState()` 的 Bundle 机制，数据会被序列化到磁盘。两者的使用原则是：用户输入型的轻量数据用 `SavedStateHandle`，可重新获取的数据（如网络请求结果）只存 ViewModel 普通字段。

### Q4: 为什么 ViewModel 不能持有 Activity/View 的引用？

**A**: ViewModel 的生命周期比 Activity 长（跨越配置变更）。如果持有旧 Activity 的引用，旧 Activity 实例无法被 GC 回收，造成内存泄漏。如果需要 Context，应使用 `AndroidViewModel`（持有 Application 引用）或通过依赖注入传入应用级 Context。

### Q5: ViewModelProvider 的 key 是怎么设计的？有什么作用？

**A**: 默认 key 格式为 `"androidx.lifecycle.ViewModelProvider.DefaultKey:全限定类名"`。它的作用是在同一个 `ViewModelStore`（HashMap）中区分不同类型的 ViewModel。当使用 `by viewModels()` 或 `ViewModelProvider(this)[MyViewModel::class.java]` 时，框架会自动生成这个 key。如果需要同一类型的多个实例，可以通过 `ViewModelProvider.get(customKey, MyViewModel::class.java)` 传入自定义 key。

### Q6: Fragment 之间如何通过 ViewModel 共享数据？原理是什么？

**A**: 多个 Fragment 调用 `by activityViewModels()` 即可获取同一个 ViewModel 实例。原理：`activityViewModels()` 内部使用 `requireActivity()` 作为 `ViewModelStoreOwner`，所有 Fragment 都从宿主 Activity 的 `ViewModelStore` 中获取 ViewModel，自然得到同一个实例。Navigation 组件还支持 `by navGraphViewModels(graphId)` 实现导航图级别的作用域共享。

### Q7: Hilt 中 @HiltViewModel 的 ViewModel 是如何被创建的？

**A**: `@HiltViewModel` + `@Inject constructor` 标注的 ViewModel，Hilt 在编译时生成对应的 `ViewModelProvider.Factory` 实现（通过 `HiltViewModelFactory`）。运行时，Hilt 拦截了 `ViewModelProvider` 的 Factory 创建过程，将 ViewModel 的构造参数（Repository、SavedStateHandle 等）从 Dagger Component 中注入。开发者只需在 Activity/Fragment 上添加 `@AndroidEntryPoint`，框架自动完成 Factory 的替换。

### Q8: ViewModel 和 onSaveInstanceState 如何配合使用？各自的职责边界是什么？

**A**: 
| 维度 | ViewModel | onSaveInstanceState (SavedStateHandle) |
|------|-----------|---------------------------------------|
| 存储介质 | 内存（ViewModelStore） | Bundle → 磁盘序列化 |
| 存活范围 | 配置变更 | 配置变更 + 进程死亡 |
| 数据大小 | 无限制（内存允许即可） | Bundle 上限约 1MB，建议极轻量 |
| 数据类型 | 任意对象 | 仅 Parcelable/Serializable/基本类型 |
| 适用数据 | 网络结果、复杂对象、大列表 | 用户输入、页面 ID、滚动位置 |

二者不是替代关系而是互补：ViewModel 处理大多数场景，SavedStateHandle 处理进程死亡后的状态恢复。推荐在 ViewModel 构造中接收 `SavedStateHandle`，将关键用户状态写入其中。
