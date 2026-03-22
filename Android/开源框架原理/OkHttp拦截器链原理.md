# OkHttp 拦截器链原理

## 一、概述

OkHttp 是 Android/Java 生态中最主流的 HTTP 客户端库，其核心设计思想就是**拦截器链（Interceptor Chain）**。所有网络请求的处理逻辑 -- 从重试、缓存到真正的网络 I/O -- 都被抽象为一个个拦截器，串成一条责任链。

这种设计的优异之处：

| 优点 | 说明 |
|------|------|
| **职责单一** | 每个拦截器只关心一件事（重试、桥接、缓存、连接、请求） |
| **可插拔** | 开发者可在链中任意位置插入自定义拦截器 |
| **顺序可控** | 拦截器的执行顺序决定了行为优先级 |
| **双向处理** | 每个拦截器既能修改 Request（去程），又能修改 Response（回程） |

> 一句话概括：OkHttp 把一次 HTTP 请求拆解成了一条流水线，每个工位（拦截器）完成自己的工作后传给下一个工位，最终拿到结果后再反向传回。

---

## 二、整体架构

### 2.1 一次请求的完整调用链

```
OkHttpClient.newCall(request)
  → RealCall.execute() / enqueue()
    → RealCall.getResponseWithInterceptorChain()
      → 构建拦截器列表
        → 创建 RealInterceptorChain
          → chain.proceed(request)
            → 依次经过所有拦截器
              → 返回 Response
```

### 2.2 拦截器的执行顺序

```
                    Request →
  ┌─────────────────────────────────────────┐
  │ 1. 用户自定义 Application Interceptors   │
  │ 2. RetryAndFollowUpInterceptor           │
  │ 3. BridgeInterceptor                     │
  │ 4. CacheInterceptor                      │
  │ 5. ConnectInterceptor                    │
  │ 6. 用户自定义 Network Interceptors        │
  │ 7. CallServerInterceptor                 │
  └─────────────────────────────────────────┘
                    ← Response
```

### 2.3 核心源码：拦截器链的构建

```kotlin
// RealCall.kt
internal fun getResponseWithInterceptorChain(): Response {
    val interceptors = mutableListOf<Interceptor>()

    // 1. 用户添加的应用拦截器（最先执行）
    interceptors += client.interceptors

    // 2~5. OkHttp 内置拦截器
    interceptors += RetryAndFollowUpInterceptor(client)
    interceptors += BridgeInterceptor(client.cookieJar)
    interceptors += CacheInterceptor(client.cache)
    interceptors += ConnectInterceptor

    // 6. 用户添加的网络拦截器
    if (!forWebSocket) {
        interceptors += client.networkInterceptors
    }

    // 7. 真正发送请求的拦截器（最后执行）
    interceptors += CallServerInterceptor(forWebSocket)

    // 构建责任链
    val chain = RealInterceptorChain(
        interceptors = interceptors,
        index = 0,
        request = originalRequest,
        // ...其他参数
    )
    val response = chain.proceed(originalRequest)
    return response
}
```

---

## 三、责任链模式的实现

### 3.1 Interceptor 接口

```kotlin
fun interface Interceptor {
    fun intercept(chain: Chain): Response

    interface Chain {
        fun request(): Request
        fun proceed(request: Request): Response
        fun connection(): Connection?
        // ...
    }
}
```

### 3.2 RealInterceptorChain -- 链的推进器

`RealInterceptorChain` 是责任链的核心实现，通过 `index` 控制当前执行到哪个拦截器：

```kotlin
class RealInterceptorChain(
    private val interceptors: List<Interceptor>,
    private val index: Int,
    private val request: Request,
    // ...
) : Interceptor.Chain {

    override fun proceed(request: Request): Response {
        // 1. 创建下一个 Chain（index + 1）
        val next = copy(index = index + 1, request = request)

        // 2. 取出当前拦截器
        val interceptor = interceptors[index]

        // 3. 调用当前拦截器，把"下一个链"传给它
        val response = interceptor.intercept(next)

        return response
    }
}
```

> 每次 `proceed()` 调用都会创建一个新的 Chain 对象（index + 1），然后把它传给当前 Interceptor。Interceptor 内部通过调用 `chain.proceed()` 把控制权传给下一个拦截器，形成递归调用。

### 3.3 双向拦截的实现原理

```kotlin
class MyInterceptor : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()

        // ========== 去程：修改 Request ==========
        val modifiedRequest = request.newBuilder()
            .addHeader("X-Custom", "value")
            .build()

        // ========== 调用下一个拦截器 ==========
        val response = chain.proceed(modifiedRequest)

        // ========== 回程：修改 Response ==========
        return response.newBuilder()
            .addHeader("X-Response-Time", "100ms")
            .build()
    }
}
```

调用栈示意（以 3 个拦截器为例）：

```
Interceptor_0.intercept(chain_1)      ← 去程
  │ chain_1.proceed(request)
  ├→ Interceptor_1.intercept(chain_2)  ← 去程
  │    │ chain_2.proceed(request)
  │    ├→ Interceptor_2.intercept(chain_3)  ← 发送请求，获取 Response
  │    │    return response
  │    ← 回到 Interceptor_1              ← 回程
  │    return response
  ← 回到 Interceptor_0                  ← 回程
  return response
```

---

## 四、五大内置拦截器详解

### 4.1 RetryAndFollowUpInterceptor -- 重试与重定向

**职责**：处理请求失败后的重试逻辑，以及 HTTP 3xx 重定向。

核心流程：

```kotlin
override fun intercept(chain: Chain): Response {
    var request = chain.request()
    var followUpCount = 0

    while (true) {
        // 准备连接（创建 Exchange）
        call.enterNetworkInterceptorExchange(request, newRoutePlanner)

        try {
            // 调用下一个拦截器
            response = realChain.proceed(request)
        } catch (e: RouteException) {
            // 路由异常 → 判断是否可重试
            if (!recover(e.lastConnectException, call, request, requestSendStarted = false)) {
                throw e.firstConnectException
            }
            continue  // 重试
        } catch (e: IOException) {
            // IO 异常 → 判断是否可重试
            if (!recover(e, call, request, requestSendStarted = e !is ConnectionShutdownException)) {
                throw e
            }
            continue  // 重试
        }

        // 处理重定向
        val followUp = followUpRequest(response, exchange)
        if (followUp == null) return response  // 无需重定向，返回

        followUpCount++
        if (followUpCount > MAX_FOLLOW_UPS) {  // 默认 20 次
            throw ProtocolException("Too many follow-up requests: $followUpCount")
        }

        request = followUp  // 用重定向的新 Request 继续循环
    }
}
```

**重试条件**（`recover()` 方法）：
- 客户端配置了 `retryOnConnectionFailure(true)`（默认开启）
- 不是协议错误（ProtocolException）
- 不是 SSL 证书异常
- 还有可用的备选路由（Route）

### 4.2 BridgeInterceptor -- 桥接拦截器

**职责**：将用户构建的 Request 补全为合规的 HTTP 请求（去程），解析压缩的 Response（回程）。

去程（补全 header）：

| 添加的 Header | 值 | 说明 |
|-------------|------|------|
| `Content-Type` | 根据 RequestBody | MIME 类型 |
| `Content-Length` / `Transfer-Encoding` | 计算或 chunked | 请求体长度 |
| `Host` | 从 URL 提取 | 目标主机 |
| `Connection` | `Keep-Alive` | 长连接 |
| `Accept-Encoding` | `gzip` | 自动请求 gzip 压缩 |
| `Cookie` | 从 CookieJar 获取 | Cookie 管理 |
| `User-Agent` | `okhttp/4.x.x` | 客户端标识 |

回程（解压）：
- 如果响应 `Content-Encoding: gzip` 且是 OkHttp 自动添加的 `Accept-Encoding`，自动用 `GzipSource` 解压
- 解压后移除 `Content-Encoding` 和 `Content-Length` header

### 4.3 CacheInterceptor -- 缓存拦截器

**职责**：实现 HTTP 缓存策略，避免不必要的网络请求。

```kotlin
override fun intercept(chain: Chain): Response {
    // 1. 从 DiskLruCache 获取缓存响应
    val cacheCandidate = cache?.get(chain.request())

    // 2. 计算缓存策略
    val strategy = CacheStrategy.Factory(now, chain.request(), cacheCandidate).compute()
    val networkRequest = strategy.networkRequest  // 需要发送的网络请求（null 表示用缓存）
    val cacheResponse = strategy.cacheResponse    // 可用的缓存响应（null 表示无缓存）

    // 3. 决策
    if (networkRequest == null && cacheResponse == null) {
        // 无网络请求，无缓存 → 504
        return Response.Builder().code(504).build()
    }

    if (networkRequest == null) {
        // 有缓存且未过期 → 直接返回缓存
        return cacheResponse!!.newBuilder().cacheResponse(cacheResponse).build()
    }

    // 4. 发起网络请求（可能带 If-None-Match / If-Modified-Since）
    val networkResponse = chain.proceed(networkRequest)

    // 5. 304 Not Modified → 用缓存 body + 网络 response header
    if (networkResponse.code == HTTP_NOT_MODIFIED) {
        val merged = cacheResponse!!.newBuilder()
            .headers(combine(cacheResponse.headers, networkResponse.headers))
            .build()
        cache?.update(cacheResponse, merged)
        return merged
    }

    // 6. 200 → 写入缓存并返回
    cache?.put(networkResponse)
    return networkResponse
}
```

缓存策略（`CacheStrategy`）遵循 HTTP/1.1 RFC 7234：
- `Cache-Control: no-cache / no-store / max-age`
- `Expires / Last-Modified / ETag`
- 条件请求：`If-None-Match`（ETag）/ `If-Modified-Since`

### 4.4 ConnectInterceptor -- 连接拦截器

**职责**：建立到目标服务器的 TCP 连接（或复用已有连接），完成 TLS 握手。

```kotlin
object ConnectInterceptor : Interceptor {
    override fun intercept(chain: Chain): Response {
        val realChain = chain as RealInterceptorChain
        val exchange = realChain.call.initExchange(realChain)
        val connectedChain = realChain.copy(exchange = exchange)
        return connectedChain.proceed(realChain.request)
    }
}
```

`initExchange()` 内部调用链：

```
RealCall.initExchange()
  → ExchangeFinder.find()
    → RealConnectionPool.callAcquirePooledConnection()  // 尝试复用连接
      → 如果没有可复用的 → 创建新 RealConnection
        → RealConnection.connect()
          → connectSocket()     // 建立 TCP 连接
          → connectTls()        // TLS 握手（HTTPS）
          → 协商 HTTP/2（ALPN）
    → 将新连接放入连接池
```

### 4.5 CallServerInterceptor -- 请求拦截器

**职责**：链中的最后一个拦截器，真正向服务器发送请求并读取响应。

```kotlin
override fun intercept(chain: Chain): Response {
    val exchange = realChain.exchange!!

    // 1. 写入请求头
    exchange.writeRequestHeaders(request)

    // 2. 写入请求体（POST/PUT 等）
    if (HttpMethod.permitsRequestBody(request.method) && requestBody != null) {
        val bufferedSink = exchange.createRequestBody(request, false).buffer()
        requestBody.writeTo(bufferedSink)
        bufferedSink.close()
    }

    // 3. 完成请求发送
    exchange.finishRequest()

    // 4. 读取响应头
    val response = exchange.readResponseHeaders(expectContinue = false)!!
        .request(request)
        .build()

    // 5. 读取响应体
    val body = exchange.openResponseBody(response)
    return response.newBuilder().body(body).build()
}
```

> 这是唯一一个**不调用 `chain.proceed()`** 的拦截器，因为它是链的终点。

---

## 五、连接池（ConnectionPool）

### 5.1 连接复用的意义

TCP 连接建立（三次握手）+ TLS 握手的开销很大。连接池通过复用已建立的连接，显著减少延迟：

| 场景 | 首次请求耗时 | 复用连接耗时 |
|------|:---:|:---:|
| HTTP | ~100ms（TCP 握手） | ~0ms |
| HTTPS | ~300ms（TCP + TLS） | ~0ms |

### 5.2 RealConnectionPool 核心

```kotlin
class RealConnectionPool(
    val maxIdleConnections: Int = 5,    // 最大空闲连接数
    val keepAliveDuration: Long = 5,    // 空闲保活时间
    val keepAliveDurationUnit: TimeUnit = TimeUnit.MINUTES
) {
    private val connections = ConcurrentLinkedQueue<RealConnection>()

    // 清理任务：定期检查并关闭过期/多余的空闲连接
    private val cleanupTask = object : Task("ConnectionPool cleanup") {
        override fun runOnce(): Long {
            return cleanup(System.nanoTime())
        }
    }
}
```

连接匹配条件（判断能否复用）：
- 相同的 `Address`（host + port + proxy + SSL 配置）
- 连接未关闭
- HTTP/2 流计数未超限

---

## 六、Application Interceptor vs Network Interceptor

```kotlin
val client = OkHttpClient.Builder()
    .addInterceptor(appInterceptor)           // Application 层
    .addNetworkInterceptor(networkInterceptor) // Network 层
    .build()
```

| 维度 | Application Interceptor | Network Interceptor |
|------|------------------------|-------------------|
| 位置 | 链的最外层（第一个） | ConnectInterceptor 之后 |
| 执行次数 | 每次 `execute/enqueue` 一次 | 重定向时每次网络请求都执行 |
| 能否短路 | 可以不调用 `proceed()`，直接返回缓存 | 必须调用 `proceed()` |
| 能否访问 Connection | 否 | 是（`chain.connection()`） |
| 看到的 Request | 用户原始 Request | Bridge 补全后的完整 Request |
| 看到重定向 | 否（只看到最终结果） | 是（每次重定向都经过） |
| 典型用途 | 统一添加 Token、日志、Mock | 压缩/解压、流量统计、修改 header |

**选择原则**：
- 业务逻辑（鉴权、日志、统一错误处理）→ Application Interceptor
- 网络层监控（流量统计、真实请求/响应观察）→ Network Interceptor

---

## 七、实战：常见自定义拦截器

### 7.1 统一添加 Token

```kotlin
class AuthInterceptor(private val tokenProvider: () -> String?) : Interceptor {
    override fun intercept(chain: Chain): Response {
        val request = chain.request()
        val token = tokenProvider()

        val newRequest = if (token != null) {
            request.newBuilder()
                .header("Authorization", "Bearer $token")
                .build()
        } else {
            request
        }
        return chain.proceed(newRequest)
    }
}
```

### 7.2 请求耗时日志

```kotlin
class LoggingInterceptor : Interceptor {
    override fun intercept(chain: Chain): Response {
        val request = chain.request()
        val startTime = System.nanoTime()

        val response = chain.proceed(request)

        val duration = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime)
        Log.d("HTTP", "${request.method} ${request.url} → ${response.code} (${duration}ms)")

        return response
    }
}
```

### 7.3 Token 过期自动刷新

```kotlin
class TokenRefreshInterceptor(
    private val authManager: AuthManager
) : Interceptor {
    override fun intercept(chain: Chain): Response {
        val response = chain.proceed(chain.request())

        if (response.code == 401) {
            synchronized(this) {
                // 刷新 Token
                val newToken = authManager.refreshTokenSync()
                if (newToken != null) {
                    // 用新 Token 重试原请求
                    val retryRequest = chain.request().newBuilder()
                        .header("Authorization", "Bearer $newToken")
                        .build()
                    response.close()
                    return chain.proceed(retryRequest)
                }
            }
        }
        return response
    }
}
```

---

## 八、常见面试题与解答

### Q1: OkHttp 的拦截器链是如何工作的？

**A**: OkHttp 采用**责任链模式**。`RealCall.getResponseWithInterceptorChain()` 将所有拦截器按顺序放入列表，创建 `RealInterceptorChain`（初始 index = 0）。每次调用 `chain.proceed()` 时，创建一个 `index + 1` 的新 Chain，取出当前 index 的拦截器执行 `intercept(nextChain)`。拦截器内部再调用 `nextChain.proceed()` 推进到下一个拦截器，形成递归调用。这使得每个拦截器可以在 `proceed()` 前修改 Request（去程），在 `proceed()` 后修改 Response（回程）。

### Q2: Application Interceptor 和 Network Interceptor 有什么区别？

**A**: Application Interceptor 在链的最外层，每次调用只执行一次，看到的是用户原始 Request；Network Interceptor 在 ConnectInterceptor 之后，看到的是 Bridge 补全后的完整 Request，且重定向时每次网络请求都会执行。Application Interceptor 可以不调用 `proceed()` 实现短路返回，适合鉴权/日志等业务逻辑；Network Interceptor 可以访问 `chain.connection()` 获取底层连接信息，适合流量统计等网络层监控。

### Q3: OkHttp 的缓存机制是怎么实现的？

**A**: `CacheInterceptor` 使用 `DiskLruCache` 存储 HTTP 响应。处理流程：先从缓存获取候选响应，通过 `CacheStrategy`（遵循 RFC 7234）计算是否可用。如果缓存有效直接返回；如果需要验证，发送带 `If-None-Match / If-Modified-Since` 的条件请求，服务器返回 304 则合并缓存和新 header 返回；否则使用网络响应并更新缓存。开发者通过 `Cache-Control` header 或 `CacheControl` 类控制缓存策略。

### Q4: OkHttp 是如何复用连接的？

**A**: `RealConnectionPool` 维护一个空闲连接队列。新请求时，`ExchangeFinder` 先在连接池中查找可复用的连接（匹配条件：相同 Address、连接未关闭、HTTP/2 流未超限）。匹配到则直接复用，无需重新 TCP/TLS 握手。默认保持最多 5 个空闲连接，空闲超过 5 分钟自动清理。对于 HTTP/2，同一 host 的多个请求可以在一个 TCP 连接上多路复用。

### Q5: OkHttp 的重试机制是怎样的？有哪些条件会触发重试？

**A**: `RetryAndFollowUpInterceptor` 在遇到 `RouteException`（路由异常）或 `IOException`（I/O 异常）时通过 `recover()` 方法判断是否重试。重试条件包括：客户端配置了 `retryOnConnectionFailure(true)`、不是协议错误、不是 SSL 证书异常、存在备选路由。重定向最多跟随 20 次（`MAX_FOLLOW_UPS`）。注意：OkHttp 不会对已发送 body 的请求（如 POST）做自动重试，因为可能导致重复操作。

### Q6: 如何实现 Token 过期后自动刷新并重试？

**A**: 在 Application Interceptor 中，先正常执行 `chain.proceed()`，检查响应状态码。如果是 401，在 `synchronized` 块中执行 Token 刷新（防止多个请求并发刷新），然后用新 Token 构建新 Request，关闭旧 Response，再次调用 `chain.proceed()` 重试。关键点：用 `synchronized` 确保只刷新一次，避免多线程竞争。

### Q7: 如果自定义拦截器中不调用 `chain.proceed()` 会怎样？

**A**: 如果是 Application Interceptor，不调用 `proceed()` 意味着短路整个链，后续拦截器（包括实际网络请求）都不会执行。这常用于 Mock 测试（直接返回预设 Response）或离线缓存策略。但如果是 Network Interceptor 不调用 `proceed()`，由于此时已经建立了连接，可能导致连接泄漏等问题，OkHttp 内部会检测并抛异常。

### Q8: OkHttp 的 BridgeInterceptor 做了什么？为什么需要它？

**A**: BridgeInterceptor 是**用户友好 API 与 HTTP 协议规范之间的桥梁**。去程：将用户构建的"简洁"Request 补全为合规的 HTTP 请求（自动添加 `Host`、`Content-Length`、`Accept-Encoding: gzip`、`Cookie`、`User-Agent` 等 header）。回程：如果响应是 gzip 压缩的（且是 OkHttp 自动请求的 gzip），自动解压并移除 `Content-Encoding` header。这让开发者无需关心 HTTP 协议细节，专注于业务逻辑。
