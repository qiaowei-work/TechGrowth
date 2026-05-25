# 限流重试机制技术文档

## 一、什么是 429 限流

### 1.1 HTTP 状态码 429

HTTP 429 (Too Many Requests) 是服务端返回的限流状态码，表示"你在短时间内发了太多请求，请慢一点"。

```
客户端                        服务端
  │                             │
  │── POST /v1/chat/completions →│
  │                             │  检查：这个客户端最近请求太多了
  │←── 429 Too Many Requests ───│
  │                             │
  │   Retry-After: 5            │  （可选）告诉你等 5 秒再试
```

### 1.2 大模型 API 的限流维度

大模型 API 的限流通常不是只看"请求次数"，而是综合看多个维度：

| 维度 | 说明 | 触发场景 |
|------|------|---------|
| QPS（每秒请求数） | 1 秒内请求太多 | 批量任务并发高 |
| 并发数 | 同时在跑的请求数太多 | 流式连接占用时间长 |
| Token 速率 | 单位时间内输入+输出 token 总量超限 | thinking 模式每步消耗大量 tokens |

### 1.3 我们的场景为什么容易触发 429

本项目使用 ReactAgent 做多步工具调用，thinking 模式下每步都是独立的模型 API 请求：

```
用户："检查 Modena 天枢数采需求开发情况"

第1步：调模型 API → thinking + tool_call("vehicleModelList")     消耗 ~3000 tokens
第2步：调模型 API → thinking + tool_call("fuzzyQueryExample")    消耗 ~3000 tokens
第3步：调模型 API → thinking + tool_call("obtainVersionIteration") 消耗 ~3000 tokens
第4步：调模型 API → thinking + tool_call("compareVersions")      消耗 ~3000 tokens
第5步：调模型 API → thinking + 最终回答                           消耗 ~5000 tokens

总计：~17000 tokens，在 1~2 分钟内集中消耗 → 触发 token 速率限流
```

---

## 二、解决思路：自动重试

429 是**临时限流**，不是永久封禁。等几秒钟，速率限制窗口刷新，请求就能通过。

核心策略：**在 HTTP 调用层拦截 429 响应，自动等待后重试**，让上层业务代码完全无感知。

```
没有重试时：
  请求 → 429 → 抛异常 → 任务失败

有重试时：
  请求 → 429 → 等 1s → 重试 → 429 → 等 2s → 重试 → 200 ✓ → 返回成功
```

---

## 三、Spring 的两套 HTTP 客户端

Spring 生态中有两套 HTTP 客户端，分别对应同步和响应式两种编程模型：

### 3.1 RestClient（同步阻塞）

```java
// 同步调用：发请求，阻塞等待响应
RestClient client = RestClient.create();
String result = client.get()
    .uri("https://api.example.com/data")
    .retrieve()
    .body(String.class);  // 阻塞在这里，直到收到响应
```

- 基于 `java.net.HttpURLConnection` 或 Apache HttpClient
- 线程模型：一个请求占一个线程，阻塞等待响应
- 拦截机制：`ClientHttpRequestInterceptor`

### 3.2 WebClient（响应式非阻塞）

```java
// 响应式调用：发请求，不阻塞，通过回调处理响应
WebClient client = WebClient.create();
Mono<String> result = client.get()
    .uri("https://api.example.com/data")
    .retrieve()
    .bodyToMono(String.class);  // 不阻塞，返回一个 Mono（未来值）

result.subscribe(data -> System.out.println(data));  // 异步处理
```

- 基于 Netty（非阻塞 IO）
- 线程模型：少量线程处理大量并发请求
- 拦截机制：`ExchangeFilterFunction`
- 适合流式（SSE）场景

### 3.3 本项目中的使用情况

```
ReactAgent
├── queryAgent.call()      → 同步调用 → 走 RestClient
└── streamingAgent.stream() → 流式调用 → 走 WebClient
```

Spring AI 内部根据调用方式选择不同的 HTTP 客户端：
- 同步 `call()` 用 RestClient
- 流式 `stream()` 用 WebClient（因为 SSE 需要非阻塞 IO）

**这就是为什么只在 RestClient 上加拦截器不够** — 流式调用走的是 WebClient，需要在 WebClient 层也加重试。

---

## 四、RestClient 拦截器机制

### 4.1 接口定义

```java
public interface ClientHttpRequestInterceptor {
    ClientHttpResponse intercept(
        HttpRequest request,              // 请求信息（URL、Header）
        byte[] body,                      // 请求体
        ClientHttpRequestExecution execution  // "放行"方法
    ) throws IOException;
}
```

### 4.2 执行流程

```
RestClient.execute()
    │
    ▼
拦截器链（按注册顺序）
    │
    ├── 拦截器A.intercept()
    │       │
    │       ├── 前置逻辑
    │       ├── execution.execute(request, body) ──→ 拦截器B / 真正的HTTP请求
    │       ├── 后置逻辑
    │       └── return response
    │
    ▼
返回最终响应
```

`execution.execute(request, body)` 有双重含义：
- 如果后面还有拦截器，调用下一个拦截器
- 如果没有了，发出真正的 HTTP 请求

**关键点**：`execution.execute()` 返回的是 `ClientHttpResponse` 对象，**不会自动抛异常**。是后续代码检查状态码后才决定是否抛异常。

### 4.3 Http429RetryInterceptor 实现

```java
public ClientHttpResponse intercept(HttpRequest request, byte[] body,
                                      ClientHttpRequestExecution execution) throws IOException {
    // 1. 先正常执行一次请求
    ClientHttpResponse response = execution.execute(request, body);

    // 2. 如果返回 429，进入重试循环
    int attempt = 0;
    while (response.getStatusCode().value() == 429 && attempt < MAX_RETRIES) {
        attempt++;
        long waitMs = calculateWaitMs(response, attempt);

        log.warn("[HTTP-429] 第 {}/{} 次重试，等待 {}ms", attempt, MAX_RETRIES, waitMs);

        response.close();       // 关闭当前响应（释放连接）
        Thread.sleep(waitMs);   // 等待
        response = execution.execute(request, body);  // 重新发请求
    }

    // 3. 返回最终响应（成功 or 仍然 429）
    return response;
}
```

### 4.4 注册方式

通过 `RestClientCustomizer` 或直接在 `RestClient.Builder` 上注册：

```java
@Bean
public RestClient.Builder restClientBuilderProvider() {
    return RestClient.builder()
            .requestInterceptor(new Http429RetryInterceptor());  // 注册拦截器
}
```

---

## 五、WebClient 过滤器机制

### 5.1 接口定义

```java
public interface ExchangeFilterFunction {
    Mono<ClientResponse> filter(ClientRequest request, ExchangeFunction next);
}
```

### 5.2 与 RestClient 拦截器的核心区别

| | RestClient 拦截器 | WebClient 过滤器 |
|--|------------------|-----------------|
| 返回类型 | `ClientHttpResponse`（同步） | `Mono<ClientResponse>`（异步） |
| 线程模型 | 阻塞，`Thread.sleep()` 可用 | 非阻塞，不能直接 sleep |
| 重试方式 | while 循环 | 递归 + `Mono.delay()` |
| 接口 | `ClientHttpRequestInterceptor` | `ExchangeFilterFunction` |

### 5.3 为什么 WebClient 不能用 Thread.sleep()

WebClient 基于 Netty 的事件循环线程，一个线程要处理成百上千个连接。如果在这个线程上 sleep，所有连接都会被阻塞。

```
Netty 事件循环线程（只有几个）
    │
    ├── 处理请求A的响应
    ├── 处理请求B的响应
    ├── 处理请求C的响应
    └── ...

如果 Thread.sleep(2000)：
    │
    ├── 处理请求A的响应
    ├── Thread.sleep(2000) ← 所有请求都卡住了！
    │
    └── 2秒后才能处理B、C...
```

所以必须用 `Mono.delay()` 来实现非阻塞等待。

### 5.4 WebClient429RetryFilter 实现

```java
public Mono<ClientResponse> filter(ClientRequest request, ExchangeFunction next) {
    return doFilter(request, next, 0);  // 从第 0 次开始
}

private Mono<ClientResponse> doFilter(ClientRequest request, ExchangeFunction next, int attempt) {
    return next.exchange(request).flatMap(response -> {
        // 检查是否 429
        if (response.statusCode().value() == 429 && attempt < MAX_RETRIES) {
            long waitMs = calculateWaitMs(response, attempt + 1);
            response.releaseBody();  // 释放连接

            // 非阻塞等待后递归重试
            return Mono.delay(Duration.ofMillis(waitMs))
                    .then(doFilter(request, next, attempt + 1));
        }
        return Mono.just(response);
    });
}
```

**递归重试流程**：

```
doFilter(request, next, 0)
    │
    ├── next.exchange(request) → 收到 429
    ├── Mono.delay(1s)         → 非阻塞等待 1 秒
    │
    └── doFilter(request, next, 1)
            │
            ├── next.exchange(request) → 收到 429
            ├── Mono.delay(2s)         → 非阻塞等待 2 秒
            │
            └── doFilter(request, next, 2)
                    │
                    └── next.exchange(request) → 收到 200 ✓
                            │
                            └── return Mono.just(response)
```

### 5.5 注册方式

WebClient 没有类似 `RestClientCustomizer` 的标准定制接口（Spring Boot 3.3.x 中不存在）。使用 `BeanPostProcessor` 拦截所有 `WebClient.Builder` Bean，注入过滤器：

```java
@Bean
public BeanPostProcessor webClientBuilderPostProcessor() {
    return new BeanPostProcessor() {
        @Override
        public Object postProcessAfterInitialization(Object bean, String beanName) {
            if (bean instanceof WebClient.Builder builder) {
                builder.filter(new WebClient429RetryFilter());  // 注入过滤器
            }
            return bean;
        }
    };
}
```

`BeanPostProcessor` 是 Spring 的标准机制：在每个 Bean 初始化完成后调用，可以对 Bean 进行修改。当检测到 Bean 类型是 `WebClient.Builder` 时，自动添加重试过滤器。

---

## 六、指数退避策略

### 6.1 什么是指数退避

每次重试的等待时间翻倍增长：1s → 2s → 4s → 8s → 16s

```java
long backoff = INITIAL_BACKOFF_MS * (1L << (attempt - 1));
// attempt=1: 1000 * 2^0 = 1000ms
// attempt=2: 1000 * 2^1 = 2000ms
// attempt=3: 1000 * 2^2 = 4000ms
// attempt=4: 1000 * 2^3 = 8000ms
// attempt=5: 1000 * 2^4 = 16000ms
```

### 6.2 为什么要指数增长

- 第 1 次重试：限流可能马上解除，短等一下
- 第 2 次重试：限流还在，需要多等一会儿
- 第 3 次重试：限流比较严，给足时间让速率窗口重置

线性增长（1s → 2s → 3s → 4s）等得太慢，指数增长更快找到合适的等待时间。

### 6.3 随机抖动（Jitter）

```java
long backoff = Math.min(INITIAL_BACKOFF_MS * (1L << (attempt - 1)), MAX_BACKOFF_MS);
return backoff + (long) (Math.random() * 500);  // 加 0~500ms 随机值
```

为什么加随机值？避免"惊群效应"：

```
没有抖动（3 个请求同时遇到 429）：
  请求A: 等 1s → 重试 ─┐
  请求B: 等 1s → 重试 ─┼→ 同时重试 → 又同时触发 429
  请求C: 等 1s → 重试 ─┘

有抖动：
  请求A: 等 1.0s → 重试
  请求B: 等 1.3s → 重试   → 错开了，不会同时重试
  请求C: 等 1.1s → 重试
```

### 6.4 上限控制

```java
Math.min(backoff, MAX_BACKOFF_MS)  // 最多等 30 秒
```

防止等待时间无限增长。5 次重试的完整时间线：

```
第1次重试：等 1.0~1.5s
第2次重试：等 2.0~2.5s
第3次重试：等 4.0~4.5s
第4次重试：等 8.0~8.5s
第5次重试：等 16.0~16.5s

最长总等待：约 32 秒
```

### 6.5 Retry-After 优先

服务端可能在 429 响应中返回 `Retry-After` header，告诉你该等多久：

```
HTTP/1.1 429 Too Many Requests
Retry-After: 10
```

优先使用服务端建议的等待时间，因为它更准确（服务端知道自己什么时候解除限流）。

---

## 七、在项目中的完整应用

### 7.1 两套重试覆盖两条路径

```
ReactAgent 调用模型 API
│
├── 同步调用（queryAgent.call）
│   └── RestClient → [Http429RetryInterceptor] → 网络
│
└── 流式调用（streamingAgent.stream）
    └── WebClient → [WebClient429RetryFilter] → 网络
```

| 组件 | 作用于 | 注册位置 | 重试方式 |
|------|--------|---------|---------|
| `Http429RetryInterceptor` | RestClient | `RestClientConfig` | `while` 循环 + `Thread.sleep()` |
| `WebClient429RetryFilter` | WebClient | `McpWebClientConfig` (BeanPostProcessor) | 递归 + `Mono.delay()` |

### 7.2 与其他拦截器的关系

```
ReactAgent ReAct 循环
│
├─ 调模型API
│   └─ HTTP 客户端
│       └─ [Http429RetryInterceptor / WebClient429RetryFilter]  ← 429 重试
│           └─ 实际 HTTP 请求
│
├─ 模型返回 tool_call
│
└─ 执行工具
    └─ [ToolCallThrottleInterceptor]  ← 工具调用间加延迟
        └─ [ToolObservabilityInterceptor]  ← 记录工具调用日志
            └─ [ToolErrorInterceptor]  ← 工具异常兜底
```

### 7.3 日志输出

重试时会输出以下日志：

```
# RestClient 层
[HTTP-429] 模型 API 限流，第 1/5 次重试，等待 1234ms | URI=http://model.mify.ai.srv/v1/chat/completions
[HTTP-429] 服务端建议等待 5s

# WebClient 层
[WebClient-429] 模型 API 限流，第 1/5 次重试，等待 1089ms | URI=http://model.mify.ai.srv/v1/chat/completions
```

---

## 八、关键文件索引

| 文件 | 路径 | 作用 |
|------|------|------|
| Http429RetryInterceptor | `llm/interceptor/Http429RetryInterceptor.java` | RestClient 429 重试 |
| WebClient429RetryFilter | `llm/interceptor/WebClient429RetryFilter.java` | WebClient 429 重试 |
| RestClientConfig | `llm/config/RestClientConfig.java` | 注册 RestClient 拦截器 |
| McpWebClientConfig | `llm/config/McpWebClientConfig.java` | 注册 WebClient 过滤器 |
