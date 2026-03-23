# Kotlin 语言特性深度解析

> 本文系统剖析 Kotlin 的核心语言特性，涵盖空安全、扩展函数、类体系、委托、内联函数、DSL 和泛型。每个特性均从设计动机出发，深入到字节码层面的实现原理，并结合 Android 实战与面试高频考点。
>
> 前置阅读：[Kotlin协程原理](Kotlin协程原理.md)、[Kotlin Flow深度解析](Kotlin%20Flow深度解析.md)

---

## 一、概述

### 1.1 Kotlin 的设计哲学

Kotlin 由 JetBrains 于 2011 年发起、2016 年正式发布，2017 年被 Google 列为 Android 一等公民，2019 年成为 **Android 官方推荐语言**。其设计目标可以概括为四个关键词：

| 关键词 | 体现 |
|--------|------|
| **安全** | 空安全类型系统，从编译期消除 NPE |
| **简洁** | 数据类、扩展函数、Lambda、类型推断，大幅减少样板代码 |
| **互操作** | 100% 兼容 Java 字节码，无缝混合调用 |
| **务实** | 不追求学术纯粹性，注重工程实用（如 `when`、`by lazy`、作用域函数） |

### 1.2 编译与运行

```
Kotlin 源码 (.kt)
    │
    ├── kotlinc ──→ JVM 字节码 (.class) ──→ JVM / Android (ART)
    ├── Kotlin/JS ──→ JavaScript
    └── Kotlin/Native ──→ LLVM IR ──→ 原生二进制
```

在 Android 中，Kotlin 编译为标准 JVM 字节码，经 D8/R8 转换为 DEX。这意味着 **Kotlin 的所有语言特性最终都要映射到 JVM 字节码能表达的范围**。理解这一点是理解后续各特性实现原理的基础 —— 空安全靠编译器检查 + 运行时断言、扩展函数靠静态方法、内联靠字节码展开。

---

## 二、空安全

空安全是 Kotlin 最具标志性的特性。根据 Google 的统计，NPE 是 Android 应用崩溃的第一大原因，Kotlin 的类型系统从根本上缓解了这一问题。

### 2.1 可空类型与非空类型

Kotlin 的类型系统将每种类型分为**非空型**和**可空型**两个层级：

```kotlin
var name: String = "Alice"    // 非空类型 — 编译器保证不为 null
var nickname: String? = null  // 可空类型 — 允许 null

name = null      // 编译错误！Non-null type String
nickname = null  // OK
name.length      // 安全，编译器已保证非空
nickname.length  // 编译错误！Only safe (?.) or non-null asserted (!!) calls allowed
```

**编译器如何保证非空？**

对于 Kotlin 自身的代码，编译器在赋值、参数传递等所有环节验证非空约束。但 Kotlin 需要和 Java 互操作，而 Java 没有空安全。因此编译器在 JVM 字节码层面插入了**运行时检查**：

```kotlin
// Kotlin 源码
fun greet(name: String) {  // 非空参数
    println("Hello, $name")
}

// 编译后的字节码（等效 Java）
public static void greet(@NotNull String name) {
    Intrinsics.checkNotNullParameter(name, "name");  // 运行时断言！
    System.out.println("Hello, " + name);
}
```

> `Intrinsics.checkNotNullParameter` 在参数为 null 时抛出 `IllegalArgumentException`（不是 `NullPointerException`），提供了更清晰的错误信息。在 Release 构建中可通过 `-Xno-param-assertions` 编译参数关闭以提升性能。

### 2.2 安全操作符详解

| 操作符 | 名称 | 说明 |
|--------|------|------|
| `?.` | 安全调用 | 对象为 null 时整个表达式返回 null，不会 NPE |
| `?:` | Elvis 运算符 | 左侧为 null 时返回右侧的默认值 |
| `!!` | 非空断言 | 强制将可空类型转为非空，为 null 时抛 NPE |
| `as?` | 安全类型转换 | 转换失败返回 null 而非抛 ClassCastException |
| `?.let { }` | 安全作用域 | 仅在非空时执行 lambda |

```kotlin
// 链式安全调用
val city = user?.address?.city  // 任一环节为 null，结果为 null

// Elvis 提供默认值
val displayName = user?.name ?: "Anonymous"

// Elvis 提前返回（Guard Clause 模式）
fun process(input: String?) {
    val data = input ?: return  // input 为 null 时直接返回
    // 此处 data 已被智能转换为 String（非空）
    println(data.length)
}

// Elvis 抛出自定义异常
val userId = intent.getStringExtra("user_id")
    ?: throw IllegalArgumentException("user_id is required")

// !! 的使用场景：确实知道不可能为 null（如 findViewById 之后）
// 但应尽可能避免，使用 requireNotNull 或 checkNotNull 替代
val view = requireNotNull(findViewById<TextView>(R.id.title)) {
    "title view must exist in this layout"
}
```

### 2.3 智能转换（Smart Cast）

Kotlin 编译器能追踪 null 检查和类型检查的结果，自动完成类型转换：

```kotlin
fun process(obj: Any?) {
    if (obj == null) return
    // 此处 obj 自动从 Any? 智能转换为 Any

    if (obj is String) {
        // 此处 obj 自动智能转换为 String
        println(obj.length)  // 无需手动 (obj as String).length
    }

    // when 表达式中的智能转换
    when (obj) {
        is Int -> println(obj + 1)          // obj 智能转换为 Int
        is String -> println(obj.uppercase()) // obj 智能转换为 String
        is List<*> -> println(obj.size)     // obj 智能转换为 List<*>
    }
}
```

**智能转换的限制**：

| 场景 | 是否支持 | 原因 |
|------|---------|------|
| 局部 `val` | 支持 | 编译器能保证检查后不会被修改 |
| 局部 `var` | 条件支持 | 检查与使用之间没有修改才行 |
| 类的 `val` 属性（无自定义 getter） | 支持 | 值确定不变 |
| 类的 `var` 属性 | 不支持 | 其他线程可能在检查后修改 |
| 类的 `val` 属性（有自定义 getter） | 不支持 | 每次调用 getter 可能返回不同值 |
| `open` 的 `val` 属性 | 不支持 | 子类可能覆写为带自定义 getter 的实现 |

### 2.4 平台类型（Platform Type）

当 Kotlin 代码调用 Java 方法时，Java 的返回值类型在 Kotlin 中表示为**平台类型**（`T!`）。平台类型意味着编译器**不知道**该值是否为 null，由开发者自行判断：

```kotlin
// Java 代码
public class JavaUtils {
    public static String getName() { return null; }
}

// Kotlin 调用
val name = JavaUtils.getName()  // 类型推断为 String!（平台类型）
println(name.length)            // 编译通过！但运行时 NPE

// 安全做法：显式声明为可空类型
val safeName: String? = JavaUtils.getName()
println(safeName?.length)       // 安全
```

**如何消除平台类型？** Java 代码可以添加**可空注解**，Kotlin 编译器能识别这些注解并据此推断准确类型：

| 注解来源 | 具体注解 |
|---------|---------|
| JetBrains | `@Nullable` / `@NotNull` |
| AndroidX | `@Nullable` / `@NonNull` |
| JSR-305 | `@javax.annotation.Nullable` / `@Nonnull` |

> **面试关键点**：平台类型是 Kotlin 空安全的**最大漏洞**。在与 Java 混编的项目中，应当在 Kotlin 侧显式标注类型（`String?`），或在 Java 侧添加可空注解，两种方式结合才能最大程度防御 NPE。

### 2.5 Kotlin Contracts（契约）

Contracts 是 Kotlin 1.3 引入的机制，允许函数向编译器提供**额外的语义信息**，帮助智能转换和控制流分析：

```kotlin
// 标准库中 require 的契约声明
public inline fun require(value: Boolean): Unit {
    contract {
        returns() implies value  // 如果函数正常返回，则 value 为 true
    }
    if (!value) throw IllegalArgumentException()
}

// 得益于契约，编译器知道 require 之后条件成立
fun process(name: String?) {
    require(name != null)
    // 此处 name 已被智能转换为 String（非空）
    println(name.length)   // 编译通过！
}
```

```kotlin
// 作用域函数的契约 — 让编译器知道 lambda 确实被执行了
public inline fun <T, R> T.let(block: (T) -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)  // block 恰好执行一次
    }
    return block(this)
}

// 得益于契约，在 let 内初始化的 val 被视为已确定赋值
val result: String
user?.let { result = it.name }  // 编译器知道 lambda 只执行一次
    ?: run { result = "default" }
println(result)  // 编译通过（如果没有契约，编译器会认为 result 可能未初始化）
```

**InvocationKind 枚举**：

| 值 | 含义 | 典型使用者 |
|----|------|-----------|
| `EXACTLY_ONCE` | 恰好执行一次 | `let`、`run`、`with`、`apply`、`also` |
| `AT_LEAST_ONCE` | 至少执行一次 | `require`、`check` |
| `AT_MOST_ONCE` | 最多执行一次 | 某些条件执行的工具函数 |
| `UNKNOWN` | 未知次数 | 默认值 |

> Contracts 目前仍是实验性 API（`@ExperimentalContracts`），但标准库中已广泛使用。自定义 Contract 只能声明在顶层函数中。

---

## 三、扩展函数与属性

扩展函数让 Kotlin 可以在**不修改类源码、不使用继承**的前提下为已有类添加新方法。这是 Kotlin 简洁性的重要来源。

### 3.1 扩展函数的本质 — 静态方法

扩展函数在字节码层面就是一个**普通的静态方法**，第一个参数是接收者对象：

```kotlin
// Kotlin 源码
fun String.addExclamation(): String {
    return this + "!"    // this 指向接收者对象
}

// 等效的 Java 字节码（反编译）
public static String addExclamation(@NotNull String $this$addExclamation) {
    Intrinsics.checkNotNullParameter($this$addExclamation, "$this$addExclamation");
    return $this$addExclamation + "!";
}
```

调用方式的转换：

```kotlin
// Kotlin 调用
"Hello".addExclamation()

// 编译后的 Java 调用
StringExtKt.addExclamation("Hello")  // 文件名Kt.方法名(接收者)
```

### 3.2 静态分发 — 扩展函数的核心限制

由于扩展函数本质是静态方法，它**不参与多态**。调用哪个扩展函数取决于**编译时**的变量类型，而非运行时的实际类型：

```kotlin
open class Animal
class Dog : Animal()

fun Animal.sound() = "..."
fun Dog.sound() = "Woof!"

fun makeSound(animal: Animal) {
    println(animal.sound())  // 始终打印 "..."，即使传入 Dog 实例
}

makeSound(Dog())  // 输出: "..."（不是 "Woof!"）
```

**原因**：`makeSound` 的参数类型是 `Animal`，编译器生成的代码调用的是 `AnimalExtKt.sound(animal)`，而非 `DogExtKt.sound(dog)`。

> **面试高频点**：扩展函数是编译时静态分发，成员函数是运行时动态分发。当扩展函数和成员函数签名冲突时，**成员函数总是优先**。

### 3.3 扩展函数 vs 成员函数

```kotlin
class MyClass {
    fun foo() = "member"  // 成员函数
}

fun MyClass.foo() = "extension"  // 扩展函数，同名同签名

MyClass().foo()  // 返回 "member" — 成员函数优先！
```

编译器会发出警告：`Extension is shadowed by a member`。这个设计保证了**类的作者对自己类的行为有最终控制权**。

### 3.4 扩展属性

扩展属性不能有 backing field（因为无法在已有类中插入新字段），所以必须通过 getter/setter 定义：

```kotlin
// 扩展属性 — 本质是静态的 getter 方法
val String.lastChar: Char
    get() = this[length - 1]

var StringBuilder.lastChar: Char
    get() = this[length - 1]
    set(value) { this.setCharAt(length - 1, value) }

// 使用
"Kotlin".lastChar  // 'n'
```

### 3.5 作用域函数（Scope Functions）

Kotlin 标准库提供了 5 个作用域函数，它们的核心差异在于**对象引用方式**和**返回值**：

| 函数 | 对象引用 | 返回值 | 典型场景 |
|------|---------|--------|---------|
| `let` | `it` | Lambda 结果 | null 检查后操作、变量作用域限制 |
| `run` | `this` | Lambda 结果 | 对象配置并计算结果 |
| `with` | `this` | Lambda 结果 | 对同一对象调用多个方法（非扩展函数形式） |
| `apply` | `this` | 对象本身 | 对象初始化/配置（Builder 模式替代） |
| `also` | `it` | 对象本身 | 附加副作用（日志、验证），不影响调用链 |

```kotlin
// let — null 安全 + 作用域限制
val length = nullableString?.let { str ->
    println("Processing: $str")
    str.length  // 返回值
}

// apply — 对象配置（最常用于 View、Bundle、Intent）
val intent = Intent(this, TargetActivity::class.java).apply {
    putExtra("key", "value")
    addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP)
}

// also — 附加副作用，不影响链式调用
fun createUser(name: String): User =
    User(name)
        .also { Log.d("UserFactory", "Created user: ${it.name}") }
        .also { analytics.trackUserCreation(it) }

// run — 配置并计算
val result = service.run {
    port = 8080
    query("SELECT ...")  // 返回查询结果
}

// with — 对已有对象做批量操作
with(binding) {
    titleText.text = data.title
    subtitleText.text = data.subtitle
    iconImage.setImageResource(data.iconRes)
}
```

**选择指南（决策树）**：

```
需要返回对象本身吗？
├── 是 → 需要用 this 访问吗？
│        ├── 是 → apply
│        └── 否 → also
└── 否 → 需要用 this 访问吗？
         ├── 是 → 是扩展函数调用吗？
         │        ├── 是 → run
         │        └── 否 → with
         └── 否 → let
```

---

## 四、类与对象特性

### 4.1 密封类与密封接口（Sealed Class / Sealed Interface）

密封类限制了继承层次 —— 所有直接子类必须在**同一编译单元**中定义（Kotlin 1.5+ 放宽为同一模块）。这是 `when` 表达式**穷举检查**的基础：

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String, val cause: Throwable? = null) : Result<Nothing>()
    data object Loading : Result<Nothing>()
}

// when 穷举 — 编译器确保覆盖所有子类，无需 else 分支
fun handleResult(result: Result<String>) = when (result) {
    is Result.Success -> showData(result.data)    // 智能转换
    is Result.Error -> showError(result.message)   // 智能转换
    Result.Loading -> showLoading()
    // 如果新增子类，编译器会在此处报错，强制处理
}
```

**密封类 vs 密封接口**（Kotlin 1.5+）：

| | 密封类 | 密封接口 |
|--|--------|---------|
| 继承方式 | 子类只能继承一个密封类 | 子类可实现多个密封接口 |
| 构造函数 | 可以有构造参数和状态 | 无构造函数 |
| 适用场景 | 带状态的有限类型集合 | 纯行为分类、多继承场景 |

```kotlin
// 密封接口 — 用于多继承场景
sealed interface UiEvent
sealed interface Trackable

data class ClickEvent(val id: String) : UiEvent, Trackable
data class ScrollEvent(val position: Int) : UiEvent
data class PageView(val url: String) : Trackable
```

**字节码层面**：密封类编译为 `abstract class`，构造函数为 `private`（防止外部继承）。`when` 的穷举检查是纯编译器行为，不产生额外运行时代码。

### 4.2 数据类（Data Class）

`data class` 让编译器自动生成以下方法，极大减少样板代码：

| 自动生成方法 | 行为 |
|-------------|------|
| `equals()` / `hashCode()` | 基于**主构造函数参数**进行结构相等比较 |
| `toString()` | 格式：`ClassName(param1=value1, param2=value2)` |
| `copy()` | 浅拷贝，可修改部分属性 |
| `componentN()` | 解构声明用（`component1()`、`component2()` ...） |

```kotlin
data class User(val name: String, val age: Int, val email: String)

val user1 = User("Alice", 30, "alice@example.com")
val user2 = user1.copy(age = 31)  // 只修改 age，其他保持不变

// 解构声明
val (name, age, email) = user1
println("$name is $age years old")

// equals 基于主构造函数参数
println(user1 == User("Alice", 30, "alice@example.com"))  // true
```

**数据类的限制与陷阱**：

| 项目 | 说明 |
|------|------|
| 主构造函数参数 | 至少一个参数，且必须是 `val` 或 `var` |
| body 中的属性 | **不参与** `equals`/`hashCode`/`toString`/`copy` |
| 继承 | 不能是 `abstract`、`open`、`sealed` 或 `inner` |
| `copy()` 是浅拷贝 | 引用类型属性共享同一对象，修改会互相影响 |

```kotlin
// 陷阱：body 中的属性不参与 equals
data class Product(val id: Int) {
    var price: Double = 0.0  // 不在主构造函数中
}

val p1 = Product(1).apply { price = 9.99 }
val p2 = Product(1).apply { price = 19.99 }
println(p1 == p2)  // true！price 不参与比较
```

> **Android 实战**：data class 是 MVI 架构中 UiState 的标配。但注意用 `copy()` 更新 StateFlow 时，必须确保所有需要触发 UI 更新的属性都在主构造函数中。

### 4.3 值类（Value Class / Inline Class）

`value class`（Kotlin 1.5+，替代之前的 `inline class`）用于包装单个值，在编译时尽可能**消除包装对象**，获得类型安全的同时避免性能开销：

```kotlin
@JvmInline
value class UserId(val value: Long)

@JvmInline
value class OrderId(val value: Long)

// 类型安全：虽然底层都是 Long，但编译器区分它们
fun findUser(id: UserId): User { ... }
fun findOrder(id: OrderId): Order { ... }

findUser(UserId(123))    // OK
findUser(OrderId(456))   // 编译错误！类型不匹配
```

**装箱与拆箱规则**：

| 场景 | 是否装箱 | 说明 |
|------|---------|------|
| 直接使用 | 不装箱 | 编译器将 `UserId` 替换为底层的 `Long` |
| 作为可空类型 `UserId?` | 装箱 | JVM 基础类型不能为 null，需要包装对象 |
| 作为泛型参数 `List<UserId>` | 装箱 | JVM 泛型擦除后需要对象 |
| 作为接口类型 | 装箱 | 接口分发需要对象 |
| `equals` / `hashCode` / `toString` | 不装箱 | 编译器直接使用底层值 |

```kotlin
// 编译前
val id: UserId = UserId(42)
println(id.value)

// 编译后（等效 Java）— 完全消除了 UserId 对象
long id = 42L;
System.out.println(id);
```

**value class 的限制**：

- 只能有**一个**主构造函数参数（Kotlin 未来可能放宽）
- 不能有 `init` 块（但可以有属性和方法）
- 不能被其他类继承（`final`）
- 可以实现接口

### 4.4 枚举类 vs 密封类

| 维度 | enum class | sealed class |
|------|-----------|-------------|
| 实例数量 | 固定有限（编译时确定所有实例） | 每个子类可以有多个实例 |
| 携带数据 | 所有枚举值结构相同 | 每个子类可以有不同的属性 |
| 状态 | 单例，无独立状态 | 每个实例可以携带独立状态 |
| 序列化 | 原生支持（by name） | 需要额外配置 |
| 性能 | 枚举值是静态常量 | 每次使用可能创建新对象（data class 子类） |
| `when` 穷举 | 支持 | 支持 |

```kotlin
// 适合 enum：有限的固定常量
enum class Direction { NORTH, SOUTH, EAST, WEST }

// 适合 sealed class：每种情况携带不同数据
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>()
    data class Error(val code: Int, val msg: String) : NetworkResult<Nothing>()
    data object Loading : NetworkResult<Nothing>()
}
```

### 4.5 object 与 companion object

**`object` 声明** — 线程安全的单例：

```kotlin
// Kotlin
object ApiClient {
    val baseUrl = "https://api.example.com"
    fun request(path: String): Response { ... }
}

// 编译后的 Java — 经典的饿汉式单例
public final class ApiClient {
    public static final ApiClient INSTANCE;
    private static final String baseUrl = "https://api.example.com";

    static {
        INSTANCE = new ApiClient();  // 类加载时创建
    }
}
```

**`companion object`** — 类的伴生对象（替代 Java 的 `static`）：

```kotlin
class User private constructor(val name: String, val email: String) {
    companion object Factory {  // 名称可省略，默认为 Companion
        fun create(name: String): User {
            return User(name, "$name@example.com")
        }
        // 伴生对象可以实现接口
        // 常用于 Fragment.newInstance() 工厂模式
    }
}

// 调用
val user = User.create("Alice")  // 看起来像静态方法
// Java 中调用: User.Factory.create("Alice") 或 User.Companion.create("Alice")
```

**编译后**：`companion object` 的方法和属性被编译为**外部类的静态方法**（如果标注了 `@JvmStatic`）或**内部类 `Companion` 的实例方法**。

```kotlin
companion object {
    @JvmStatic  // 生成真正的 Java 静态方法
    fun create(name: String): User { ... }
}
```

> **面试关键点**：`object` 是实例级别的单例（类加载时初始化），`companion object` 是类级别的静态容器（本质是一个嵌套的 `object`，每个类最多一个）。

---

## 五、委托

Kotlin 的委托机制允许将工作**转交给其他对象**，分为**类委托**和**属性委托**两种。

### 5.1 类委托（Class Delegation）

通过 `by` 关键字将接口的实现委托给另一个对象，避免手写大量转发代码：

```kotlin
interface Logger {
    fun log(message: String)
    fun error(message: String)
}

class ConsoleLogger : Logger {
    override fun log(message: String) = println("[LOG] $message")
    override fun error(message: String) = println("[ERROR] $message")
}

// 通过 by 将 Logger 的实现全部委托给 consoleLogger
class TaggedLogger(
    private val tag: String,
    private val delegate: Logger
) : Logger by delegate {
    // 只覆写需要定制的方法
    override fun log(message: String) {
        delegate.log("[$tag] $message")  // 加上 tag 前缀
    }
    // error() 方法自动委托给 delegate，无需手写
}
```

**字节码层面**：编译器为类生成所有未覆写的接口方法，方法体中直接调用委托对象的对应方法。本质是编译器自动生成的**装饰器模式**。

### 5.2 属性委托基础

属性委托通过 `by` 将属性的 `get`/`set` 委托给一个**委托对象**。委托对象必须提供 `getValue`（和可选的 `setValue`）操作符方法：

```kotlin
// 委托协议（operator 约定）
operator fun getValue(thisRef: Any?, property: KProperty<*>): T
operator fun setValue(thisRef: Any?, property: KProperty<*>, value: T)
```

### 5.3 by lazy — 延迟初始化

`by lazy` 是最常用的属性委托，实现了**线程安全的延迟初始化**：

```kotlin
val heavyObject: HeavyObject by lazy {
    println("Initializing...")
    HeavyObject()  // 首次访问时执行，结果被缓存
}
```

**三种线程安全模式**：

| 模式 | 线程安全 | 性能 | 实现方式 |
|------|---------|------|---------|
| `LazyThreadSafetyMode.SYNCHRONIZED`（默认） | 安全 | 有锁开销 | **双重检查锁**（DCL） |
| `LazyThreadSafetyMode.PUBLICATION` | 安全 | 可能多次初始化，但只有一个值被使用 | CAS |
| `LazyThreadSafetyMode.NONE` | 不安全 | 最快 | 无同步 |

```kotlin
// 默认模式的底层实现（简化）
private class SynchronizedLazyImpl<out T>(initializer: () -> T) : Lazy<T> {
    private var initializer: (() -> T)? = initializer
    @Volatile private var _value: Any? = UNINITIALIZED

    override val value: T
        get() {
            val v1 = _value
            if (v1 !== UNINITIALIZED) return v1 as T  // 快速路径：已初始化

            synchronized(this) {                       // 慢速路径：加锁
                val v2 = _value
                if (v2 !== UNINITIALIZED) return v2 as T  // 双重检查
                val typedValue = initializer!!()
                _value = typedValue
                initializer = null                      // 释放 initializer 引用
                return typedValue
            }
        }
}
```

> **Android 实战**：在 Activity/Fragment 中使用 `by lazy` 初始化 View Binding 或 ViewModel 时，可以用 `NONE` 模式（因为这些属性只在主线程访问）：`by lazy(LazyThreadSafetyMode.NONE) { ... }`

### 5.4 Delegates.observable 与 vetoable

```kotlin
// observable — 属性变化时触发回调
var username: String by Delegates.observable("Guest") { prop, old, new ->
    println("$old -> $new")
    notifyListeners(new)  // 适合实现简单的观察者模式
}

// vetoable — 属性变化前拦截，返回 false 拒绝修改
var age: Int by Delegates.vetoable(0) { prop, old, new ->
    new >= 0  // 只接受非负值
}

age = 25   // OK，new = 25 >= 0
age = -1   // 被拒绝，age 仍为 25
```

### 5.5 by map — 从 Map 中读取属性

将属性的值委托给 Map 的对应 key：

```kotlin
class Config(private val map: Map<String, Any?>) {
    val name: String by map        // 从 map["name"] 读取
    val version: Int by map        // 从 map["version"] 读取
    val debug: Boolean by map      // 从 map["debug"] 读取
}

val config = Config(mapOf(
    "name" to "MyApp",
    "version" to 3,
    "debug" to true
))
println(config.name)  // "MyApp"
```

> 这个模式在解析 JSON 配置或 Bundle 参数时特别有用。

### 5.6 自定义属性委托

```kotlin
// 自定义：SharedPreferences 委托
class SharedPreferencesDelegate<T>(
    private val prefs: SharedPreferences,
    private val key: String,
    private val defaultValue: T
) : ReadWriteProperty<Any?, T> {

    @Suppress("UNCHECKED_CAST")
    override fun getValue(thisRef: Any?, property: KProperty<*>): T {
        return when (defaultValue) {
            is String -> prefs.getString(key, defaultValue) as T
            is Int -> prefs.getInt(key, defaultValue) as T
            is Boolean -> prefs.getBoolean(key, defaultValue) as T
            is Long -> prefs.getLong(key, defaultValue) as T
            is Float -> prefs.getFloat(key, defaultValue) as T
            else -> throw IllegalArgumentException("Unsupported type")
        }
    }

    override fun setValue(thisRef: Any?, property: KProperty<*>, value: T) {
        prefs.edit().apply {
            when (value) {
                is String -> putString(key, value)
                is Int -> putInt(key, value)
                is Boolean -> putBoolean(key, value)
                is Long -> putLong(key, value)
                is Float -> putFloat(key, value)
            }
            apply()
        }
    }
}

// 使用
var userName: String by SharedPreferencesDelegate(prefs, "user_name", "")
userName = "Alice"          // 自动写入 SP
println(userName)           // 自动从 SP 读取
```

### 5.7 provideDelegate — 委托创建时的拦截

`provideDelegate` 操作符允许在委托被创建时进行**验证或自定义逻辑**：

```kotlin
class ResourceLoader {
    operator fun provideDelegate(
        thisRef: Any?,
        property: KProperty<*>
    ): ReadOnlyProperty<Any?, String> {
        // 在委托创建时验证属性名是否合法
        check(property.name.startsWith("res_")) {
            "Property name must start with 'res_'"
        }
        return ReadOnlyProperty { _, prop -> loadResource(prop.name) }
    }
}

val res_icon: String by ResourceLoader()    // OK
// val myIcon: String by ResourceLoader()   // 运行时报错：不以 res_ 开头
```

---

## 六、内联函数

### 6.1 inline 的本质 — 字节码展开

`inline` 关键字让编译器将函数体和传入的 Lambda **内联展开**到调用处，消除函数调用和 Lambda 对象的开销：

```kotlin
// 定义
inline fun measureTime(block: () -> Unit): Long {
    val start = System.nanoTime()
    block()
    return System.nanoTime() - start
}

// 调用
val time = measureTime {
    Thread.sleep(100)
}

// 编译后（等效 Java）— 完全展开，无函数调用、无 Lambda 对象
long start = System.nanoTime();
Thread.sleep(100);                    // block 的内容直接内联
long time = System.nanoTime() - start;
```

**没有 inline 时的开销**：每次调用高阶函数时，编译器会创建一个**匿名内部类**（或使用 `invokedynamic`）来包装 Lambda，存在以下成本：

| 开销 | 说明 |
|------|------|
| 对象创建 | 每次调用创建 Function 对象（如果 Lambda 捕获了变量） |
| 方法调用 | 通过 `Function.invoke()` 间接调用 |
| GC 压力 | 短生命周期的 Lambda 对象增加 GC 工作量 |

> inline 消除了上述所有开销，代价是**代码膨胀** —— 每个调用处都复制一份函数体。因此 inline 适合**短小的高阶函数**（如标准库的 `let`、`run`、`forEach`），不适合大函数体。

### 6.2 noinline — 禁止内联特定 Lambda

当 inline 函数接收多个 Lambda 参数时，某些参数可能需要**保留为对象**（如需要存储或传递给非 inline 函数）：

```kotlin
inline fun execute(
    inlinedBlock: () -> Unit,
    noinline storedBlock: () -> Unit  // 不内联，保留为 Function 对象
) {
    inlinedBlock()                     // 内联展开
    saveForLater(storedBlock)          // 需要对象引用才能存储
}
```

### 6.3 crossinline — 禁止非局部返回

内联 Lambda 可以使用 `return` 从**外层函数**返回（非局部返回）。`crossinline` 禁止这种行为：

```kotlin
inline fun runOnUiThread(crossinline block: () -> Unit) {
    handler.post {
        // block 在另一个 Lambda 中执行
        // crossinline 防止 block 内部使用 return 导致意外行为
        block()
    }
}

// 非局部返回 vs 局部返回
inline fun findFirst(list: List<Int>, predicate: (Int) -> Boolean): Int? {
    list.forEach {             // forEach 也是 inline
        if (predicate(it)) {
            return it          // 非局部返回：直接从 findFirst 返回
        }
    }
    return null
}

// 局部返回 — 使用标签
list.forEach {
    if (it < 0) return@forEach  // 只跳过当前迭代，不退出外层函数
}
```

**三者关系**：

| 修饰符 | Lambda 是否内联 | 是否允许非局部 return | 使用场景 |
|--------|---------------|--------------------|---------|
| 无修饰（默认） | 内联 | 允许 | 大多数场景 |
| `noinline` | 不内联 | 不允许 | 需要存储 Lambda 引用 |
| `crossinline` | 内联 | 不允许 | Lambda 在另一个执行上下文中调用 |

### 6.4 reified — 具体化类型参数

JVM 的泛型是通过**类型擦除**实现的，运行时无法获取泛型的实际类型。`inline` + `reified` 突破了这个限制：

```kotlin
// 没有 reified — 必须传 Class 参数
fun <T> Bundle.getParcelable(key: String, clazz: Class<T>): T? {
    return getParcelable(key) as? T
}
intent.extras?.getParcelable("user", User::class.java)

// 有 reified — 编译器在调用处插入实际类型
inline fun <reified T> Bundle.getParcelableCompat(key: String): T? {
    return if (Build.VERSION.SDK_INT >= 33) {
        getParcelable(key, T::class.java)  // reified 使得 T::class 合法
    } else {
        @Suppress("DEPRECATION")
        getParcelable(key) as? T
    }
}
intent.extras?.getParcelableCompat<User>("user")  // 更简洁
```

**reified 的常见用途**：

| 场景 | 示例 |
|------|------|
| 获取 Class 对象 | `T::class.java` |
| 类型判断 | `value is T` |
| 类型转换 | `value as T` |
| 创建 Intent | `inline fun <reified T : Activity> Context.startActivity()` |
| JSON 反序列化 | `inline fun <reified T> String.fromJson(): T` |

```kotlin
// 实用封装：简化 Activity 启动
inline fun <reified T : Activity> Context.startActivity(
    vararg extras: Pair<String, Any?>
) {
    val intent = Intent(this, T::class.java)
    extras.forEach { (key, value) ->
        when (value) {
            is String -> intent.putExtra(key, value)
            is Int -> intent.putExtra(key, value)
            is Boolean -> intent.putExtra(key, value)
            is Parcelable -> intent.putExtra(key, value)
        }
    }
    startActivity(intent)
}

// 调用
startActivity<DetailActivity>("id" to 42, "title" to "Hello")
```

### 6.5 内联函数的使用边界

| 适合 inline | 不适合 inline |
|------------|-------------|
| 短小的高阶函数（`let`、`forEach`） | 函数体很长（代码膨胀） |
| 需要 `reified` 类型参数 | 无 Lambda 参数的普通函数（没有对象开销可消除） |
| 性能敏感路径上的 Lambda | 递归函数（无法内联递归调用） |
| 需要非局部返回的场景 | public API 中的大函数（每次修改会导致所有调用处重新编译） |

---

## 七、DSL 构建

Kotlin 的多个语言特性组合在一起，能够创建**类型安全的领域特定语言（DSL）**。理解 DSL 构建机制是理解 Gradle Kotlin DSL、Jetpack Compose 和很多 Android 库 API 的基础。

### 7.1 Lambda with Receiver（带接收者的 Lambda）

这是 DSL 构建的核心机制。带接收者的 Lambda 允许在 Lambda 体内**直接调用接收者的方法**，无需显式引用：

```kotlin
// 普通 Lambda
val greet: (StringBuilder) -> Unit = { sb ->
    sb.append("Hello")
    sb.append(" World")
}

// 带接收者的 Lambda — StringBuilder 成为 this
val greetDsl: StringBuilder.() -> Unit = {
    append("Hello")     // this.append("Hello")
    append(" World")    // this.append(" World")
}

// 调用方式
val sb = StringBuilder()
sb.greetDsl()           // 或 greetDsl(sb)
println(sb)             // "Hello World"
```

**这就是 `apply` 的实现原理**：

```kotlin
public inline fun <T> T.apply(block: T.() -> Unit): T {
    block()    // this（接收者）调用带接收者的 Lambda
    return this
}
```

### 7.2 构建类型安全的 DSL

结合扩展函数、Lambda with receiver 和运算符重载，可以创建直观的 DSL：

```kotlin
// 定义 HTML DSL
class HTML {
    private val children = mutableListOf<Element>()

    fun head(init: Head.() -> Unit) {
        children.add(Head().apply(init))
    }

    fun body(init: Body.() -> Unit) {
        children.add(Body().apply(init))
    }

    override fun toString() = children.joinToString("\n")
}

class Head : Element() {
    fun title(text: String) { /* ... */ }
    fun meta(charset: String) { /* ... */ }
}

class Body : Element() {
    fun h1(text: String) { /* ... */ }
    fun p(text: String) { /* ... */ }
    fun div(init: Body.() -> Unit) { /* 嵌套 */ }
}

fun html(init: HTML.() -> Unit): HTML = HTML().apply(init)

// 使用 DSL — 看起来像声明式语法
val page = html {
    head {
        title("My Page")
        meta("UTF-8")
    }
    body {
        h1("Welcome")
        p("This is a paragraph")
        div {
            p("Nested content")
        }
    }
}
```

### 7.3 @DslMarker — 作用域控制

没有 `@DslMarker` 时，嵌套 Lambda 可以访问外层接收者的方法，造成混淆：

```kotlin
// 问题：在 body 的 lambda 中可以调用 head() — 这是不合理的
html {
    body {
        head { }  // 编译通过！但语义上错误
    }
}
```

`@DslMarker` 注解限制了**同一 DSL 标记的接收者不能隐式嵌套访问**：

```kotlin
@DslMarker
annotation class HtmlDsl

@HtmlDsl class HTML { ... }
@HtmlDsl class Head : Element() { ... }
@HtmlDsl class Body : Element() { ... }

// 现在：
html {
    body {
        head { }  // 编译错误！HtmlDsl 作用域限制
        this@html.head { }  // 必须显式指定外层接收者
    }
}
```

### 7.4 实际应用中的 DSL

| DSL | 底层机制 |
|-----|---------|
| **Gradle Kotlin DSL** | `dependencies { }` 使用 `DependencyHandler.() -> Unit` |
| **Jetpack Compose** | `@Composable` 函数 + `Modifier` 链 + `@DslMarker` |
| **Ktor** | `routing { }`, `get { }` 使用嵌套 Lambda with receiver |
| **kotlinx.html** | 完整的 HTML 构建 DSL |
| **MockK** | `every { mock.method() } returns value` |

```kotlin
// Gradle Kotlin DSL 示例
dependencies {
    // this 是 DependencyHandler
    implementation("com.google.android.material:material:1.9.0")
    testImplementation("junit:junit:4.13.2")
}

// Compose 示例 — @Composable Lambda with receiver
Column(
    modifier = Modifier
        .fillMaxWidth()
        .padding(16.dp)
) {
    // this 是 ColumnScope
    Text("Title", modifier = Modifier.align(Alignment.CenterHorizontally))
}
```

---

## 八、泛型与型变

### 8.1 类型擦除与 JVM 限制

和 Java 一样，Kotlin 的泛型在 JVM 上通过**类型擦除**实现 —— 运行时 `List<String>` 和 `List<Int>` 是同一个类 `List`：

```kotlin
val strings: List<String> = listOf("a", "b")
val ints: List<Int> = listOf(1, 2)

// 运行时无法区分泛型类型
println(strings is List<String>)  // 编译错误！Cannot check for erased type
println(strings is List<*>)      // OK — 星投影不检查类型参数

// reified 突破擦除（见第六章）
inline fun <reified T> checkType(value: Any) = value is T
```

### 8.2 声明处型变（Declaration-site Variance）

Kotlin 使用 `out`（协变）和 `in`（逆变）关键字在**类型声明处**指定型变，替代 Java 的使用处通配符：

```kotlin
// out — 协变：T 只能作为输出（返回值），不能作为输入（参数）
interface Producer<out T> {
    fun produce(): T        // OK — T 在 out 位置
    // fun consume(item: T) // 编译错误！T 不能在 in 位置
}

// in — 逆变：T 只能作为输入（参数），不能作为输出（返回值）
interface Consumer<in T> {
    fun consume(item: T)    // OK — T 在 in 位置
    // fun produce(): T     // 编译错误！T 不能在 out 位置
}
```

**型变规则**：

| 关键字 | 含义 | 子类型方向 | T 的位置限制 | Java 等价 |
|--------|------|-----------|-------------|-----------|
| `out T` | 协变 | `Producer<Dog>` 是 `Producer<Animal>` 的子类型 | 只能出现在返回值/val 属性 | `? extends T` |
| `in T` | 逆变 | `Consumer<Animal>` 是 `Consumer<Dog>` 的子类型 | 只能出现在参数 | `? super T` |
| 无修饰 | 不变 | `Box<Dog>` 和 `Box<Animal>` 无关系 | 任意位置 | `T` |

```kotlin
// 协变实例 — List<out E> 声明为协变
val dogs: List<Dog> = listOf(Dog(), Dog())
val animals: List<Animal> = dogs  // OK！List<Dog> 是 List<Animal> 的子类型
// 因为 List 是只读的（只有 get，没有 add），所以协变是安全的

// 逆变实例 — Comparable<in T>
val animalComparator: Comparable<Animal> = ...
val dogComparator: Comparable<Dog> = animalComparator  // OK！
// 能比较 Animal 的比较器，自然也能比较 Dog
```

### 8.3 使用处型变（Use-site Variance / Type Projection）

当类本身不是协变/逆变的，但在特定使用场景下需要型变时：

```kotlin
// Array<T> 是不变的（既能读又能写）
// 但在某些场景下，我们只需要读取
fun copyOut(from: Array<out Animal>, to: Array<Animal>) {
    for (i in from.indices) {
        to[i] = from[i]     // from 被声明为 out → 只能读取
    }
}

val dogs = arrayOf(Dog(), Dog())
val animals = arrayOf<Animal>(Cat(), Cat())
copyOut(dogs, animals)  // Array<Dog> → Array<out Animal>，合法
```

### 8.4 星投影（Star Projection）

当不关心具体类型参数时，使用 `*`：

```kotlin
// * 等价于 out Any?（对于协变和不变参数）
fun printAll(list: List<*>) {
    list.forEach { println(it) }  // 元素类型为 Any?
}

// 对于 in/out 位置的不同含义
// MutableList<*> ≈ MutableList<out Any?> as producer + MutableList<in Nothing> as consumer
val list: MutableList<*> = mutableListOf(1, "two", 3.0)
val item = list[0]      // OK — 返回 Any?
// list.add("new")      // 编译错误！无法确定类型安全
```

### 8.5 与 Java PECS 对比

| 概念 | Java | Kotlin |
|------|------|--------|
| 协变（Producer） | `List<? extends Animal>` | `List<out Animal>` 或声明处 `List<out E>` |
| 逆变（Consumer） | `List<? super Dog>` | `List<in Dog>` 或声明处 `Comparable<in T>` |
| 不变 | `List<Animal>` | `MutableList<Animal>` |
| 通配符 | `List<?>` | `List<*>` |
| 原则 | PECS（Producer Extends, Consumer Super） | 声明处用 `out`/`in`，使用处用投影 |
| 具体化泛型 | 不支持 | `inline` + `reified` |

> **Kotlin 的优势**：声明处型变将型变约束放在类定义处（如 `List<out E>`），调用方无需每次写通配符。Java 的 `List<? extends E>` 必须在每个使用处重复声明，冗余且容易遗忘。

### 8.6 where 子句 — 多重上界约束

```kotlin
// 泛型参数同时满足多个约束
fun <T> copyIfComparable(list: List<T>, target: MutableList<T>)
    where T : Comparable<T>,    // T 必须可比较
          T : Serializable {    // T 必须可序列化
    list.filter { it > list.first() }
        .forEach { target.add(it) }
}

// Java 等价: <T extends Comparable<T> & Serializable>
```

---

## 九、常见面试题与解答

### Q1：Kotlin 的空安全是如何实现的？为什么还是可能出现 NPE？

**答**：

Kotlin 的空安全在**两个层面**实现：

1. **编译期**：类型系统区分 `T`（非空）和 `T?`（可空），编译器在类型检查阶段拒绝所有可能产生 NPE 的操作
2. **运行时**：对非空参数插入 `Intrinsics.checkNotNullParameter()` 断言，防止 Java 传入 null

但 NPE 仍可能出现的场景：
- **`!!` 操作符**：强制转换可空类型为非空，空时抛 NPE
- **平台类型**：调用 Java 方法返回的 `T!` 类型，编译器不检查
- **未初始化的 `lateinit var`**：访问未初始化的属性抛 `UninitializedPropertyAccessException`
- **泛型类型擦除**：运行时无法检查泛型参数
- **Java 互操作**：Java 代码不受 Kotlin 空安全约束

---

### Q2：扩展函数的本质是什么？为什么它不能实现多态？

**答**：

扩展函数在字节码层面是**静态方法**，第一个参数是接收者对象。调用 `"hello".myFun()` 编译为 `MyFileKt.myFun("hello")`。

不能实现多态是因为静态方法的**分发在编译时确定**（基于变量声明类型），而非运行时（基于实际对象类型）。这与 Java 的 `static` 方法一样。

当扩展函数与成员函数签名冲突时，**成员函数总是优先**。此外，扩展函数无法访问类的 `private` 或 `protected` 成员。

---

### Q3：data class 自动生成了哪些方法？有什么陷阱？

**答**：

自动生成 `equals()`、`hashCode()`、`toString()`、`copy()`、`componentN()`（用于解构）。

核心陷阱：
1. **只基于主构造函数参数**：定义在 body 中的属性不参与这些方法
2. **`copy()` 是浅拷贝**：引用类型属性共享同一对象
3. **用作 `StateFlow` 的值时**：如果修改了同一个对象再赋值，`equals` 可能返回 true，导致 UI 不更新

---

### Q4：sealed class 和 enum class 如何选择？

**答**：

- **enum**：所有实例结构相同、数量固定、无独立状态（如方向、状态码）
- **sealed class**：每个子类结构不同、可携带独立数据（如 `Success(data)` / `Error(msg)` / `Loading`）

两者都支持 `when` 穷举检查。实际开发中常见的 `Result<T>` / `UiState` 模型用 sealed class，简单的常量枚举用 enum。

sealed class 编译为 `abstract class`，构造函数为 `private`。Kotlin 1.5+ 还引入了 sealed interface，支持多继承。

---

### Q5：by lazy 的线程安全是如何实现的？有几种模式？

**答**：

三种模式：

1. **`SYNCHRONIZED`**（默认）：经典双重检查锁（DCL）。`@Volatile` 标记 `_value`，首次访问 `synchronized(this)` 加锁初始化
2. **`PUBLICATION`**：多线程可能同时初始化，但通过 CAS 保证最终只有一个值被采纳（适合初始化无副作用的场景）
3. **`NONE`**：无同步，最快但不安全（适合确保单线程访问的场景，如主线程的 View Binding）

`SYNCHRONIZED` 的要点：`@Volatile` 保证可见性，`synchronized` 保证原子性，双重检查避免每次访问都加锁。

---

### Q6：inline、noinline、crossinline 的区别和使用场景？

**答**：

| | inline | noinline | crossinline |
|--|--------|----------|-------------|
| Lambda 是否内联 | 是 | 否 | 是 |
| 允许非局部 return | 是 | 否 | 否 |
| 使用场景 | 默认行为 | Lambda 需要作为对象存储 | Lambda 在其他执行上下文中调用 |

`inline` 消除 Lambda 对象开销（展开到调用处），适合短小高阶函数。`noinline` 保留 Lambda 为对象（用于 `saveForLater(block)`）。`crossinline` 禁止非局部返回（用于 `handler.post { block() }` 这种跨上下文场景）。

配合 `reified` 可在运行时获取泛型类型（`T::class.java`），突破 JVM 类型擦除。

---

### Q7：Kotlin 的 DSL 是如何实现的？@DslMarker 解决什么问题？

**答**：

DSL 构建的核心是 **Lambda with receiver**（`T.() -> Unit`），让 Lambda 内部以 `this` 访问接收者的方法，创造出声明式的语法体验。配合扩展函数和中缀函数，可以构造出类型安全的 DSL。

`@DslMarker` 解决**作用域泄漏**问题：多层嵌套的 Lambda with receiver 中，内层可以隐式访问外层接收者的方法，造成语义错误（如在 `body { }` 中能调用 `head { }`）。标注了 `@DslMarker` 后，同一标记的嵌套接收者必须显式指定（`this@outer.method()`）。

Gradle Kotlin DSL、Jetpack Compose 的 `ColumnScope`/`RowScope`、Ktor 路由都是这个机制的实际应用。

---

### Q8：Kotlin 的 out 和 in 对应 Java 的什么？为什么 List 是协变的？

**答**：

- `out T` 对应 Java 的 `? extends T`（协变，PECS 中的 Producer）
- `in T` 对应 Java 的 `? super T`（逆变，PECS 中的 Consumer）

Kotlin 的 `List<out E>` 是**只读**的（只有 `get`，没有 `add`），类型参数 `E` 只出现在输出位置，因此协变是安全的：`List<Dog>` 可以赋给 `List<Animal>`。

而 `MutableList<E>` 是不变的（既读又写），不能协变。如果允许 `MutableList<Dog>` 赋给 `MutableList<Animal>`，就可以通过后者往列表里放 `Cat`，破坏类型安全。

Kotlin 的优势是**声明处型变**，在类定义处一次声明 `out`/`in`，调用方自动获得正确的子类型关系，无需像 Java 那样每处都写通配符。

---

### Q9：Kotlin 和 Java 互操作时有哪些注意事项？

**答**：

| 注意事项 | 说明 |
|---------|------|
| **平台类型** | Java 方法返回值在 Kotlin 中是 `T!`，需显式标注为 `T?` 或 `T` |
| **可空注解** | Java 侧添加 `@Nullable`/`@NonNull` 可消除平台类型 |
| **`@JvmStatic`** | `companion object` 的方法需标注后 Java 才能直接调用 |
| **`@JvmField`** | 去除 getter/setter，暴露为 Java public field |
| **`@JvmOverloads`** | 为有默认参数的函数生成重载版本 |
| **`@JvmName`** | 解决类型擦除后的签名冲突 |
| **SAM 转换** | Kotlin Lambda 可自动转换为 Java 的单方法接口 |
| **属性访问** | Java 通过 `getXxx()`/`setXxx()` 访问 Kotlin 属性 |
| **`@Throws`** | Kotlin 不强制检查异常，Java 调用时需要此注解才能 catch |

---

### Q10：Kotlin 中 let、run、with、apply、also 如何选择？

**答**：

| 函数 | 对象引用 | 返回值 | 最佳场景 |
|------|---------|--------|---------|
| `let` | `it` | Lambda 结果 | null 安全 `?.let { }`、链式变换 |
| `run` | `this` | Lambda 结果 | 配置对象并返回计算结果 |
| `with` | `this` | Lambda 结果 | 对已有对象批量操作（非扩展函数） |
| `apply` | `this` | 对象本身 | 对象初始化/Builder 替代 |
| `also` | `it` | 对象本身 | 附加副作用（日志、验证） |

核心区别两个维度：**this vs it**（`this` 适合对对象大量操作，`it` 适合作为参数传递或避免 `this` 遮蔽）和**返回 Lambda 结果 vs 返回对象本身**（返回自身适合链式构建，返回结果适合变换）。

实际使用频率：`apply` > `let` > `also` > `run` > `with`。`apply` 在 Android 中用于 Intent/Bundle/Paint 等对象的初始化极其常见。




