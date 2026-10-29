---
tags: [java, 中段, SpringBoot, Web, 分层, 事务, 部署]
type: guide
created: 2026-10-21
updated: 2026-10-21
---

# Web 开发与分层实践

> **什么时候该读**：要写一个完整的 Spring Boot 服务；或者想把「能跑」的项目整理成「能维护」的结构。

## 学习目标

读完本篇，你应当能够：

1. 画出 Controller / Service / Repository 三层职责边界
2. 写出一组标准 REST 接口（含参数校验、统一响应、全局异常）
3. 正确使用声明式事务与连接池
4. 打包、配置分环境、开可观测性端点，并把服务跑进容器

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[java/02-中段阶段-专题精讲/SpringBoot/启动原理与自动配置]] | 启动与自动配置 |
| [[java/02-中段阶段-专题精讲/SpringBoot/IoC与Bean生命周期]] | Bean、AOP、`@Transactional` 失效场景 |

---

## 一、三层职责边界

```
HTTP 请求
   ↓
┌─────────────────────────────────┐
│ Controller（接口层）             │ 参数校验、协议转换、返回响应
│  不写业务逻辑、不直接操作数据库    │
└─────────────────────────────────┘
   ↓ DTO
┌─────────────────────────────────┐
│ Service（业务层）                │ 业务规则、事务边界、组合多个 Repository
│  ⭐ 事务加在这一层                │
└─────────────────────────────────┘
   ↓ Entity
┌─────────────────────────────────┐
│ Repository（数据访问层）          │ 只做 CRUD，不含业务判断
└─────────────────────────────────┘
   ↓
数据库
```

⭐ **一条铁律**：事务边界在 **Service 层**——Controller 不开事务、Repository 不组合事务。

### 三种对象的区别（避免混乱）

| 对象 | 用途 | 例子 |
|------|------|------|
| **DTO** | 接收外部输入 | `CreateOrderRequest` |
| **VO** | 返回给前端 | `OrderVO` |
| **Entity** | 映射数据库表 | `Order` |

⚠️ **不要把 Entity 直接返回给前端**——会泄露表结构、带上敏感字段、且无法独立演进。

## 二、一组标准 REST 接口

```java
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor          // Lombok：为 final 字段生成构造器
public class OrderController {

    private final OrderService orderService;

    @GetMapping("/{id}")
    public Result<OrderVO> get(@PathVariable Long id) {
        return Result.ok(orderService.get(id));
    }

    @GetMapping
    public Result<PageResult<OrderVO>> list(
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "20") int size) {
        return Result.ok(orderService.list(page, size));
    }

    @PostMapping
    public Result<Long> create(@RequestBody @Valid CreateOrderRequest req) {
        return Result.ok(orderService.create(req));
    }

    @PutMapping("/{id}")
    public Result<Void> update(@PathVariable Long id,
                               @RequestBody @Valid UpdateOrderRequest req) {
        orderService.update(id, req);
        return Result.ok();
    }

    @DeleteMapping("/{id}")
    public Result<Void> delete(@PathVariable Long id) {
        orderService.delete(id);
        return Result.ok();
    }
}
```

### 注解速查

| 注解 | 作用 |
|------|------|
| `@RestController` | `@Controller` + `@ResponseBody`，返回 JSON |
| `@RequestMapping` / `@GetMapping` 等 | 路由映射 |
| `@PathVariable` | 路径参数 `/orders/{id}` |
| `@RequestParam` | 查询参数 `?page=1` |
| `@RequestBody` | 请求体 JSON → 对象 |
| `@Valid` | 触发 Bean Validation 校验 |

## 三、统一响应与全局异常

### 统一响应结构

```java
public record Result<T>(int code, String message, T data) {
    public static <T> Result<T> ok(T data) { return new Result<>(0, "ok", data); }
    public static <T> Result<T> fail(int code, String msg) { return new Result<>(code, msg, null); }
}
```

### 全局异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BizException.class)
    public Result<Void> handleBiz(BizException e) {
        return Result.fail(e.getCode(), e.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleValid(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
            .map(f -> f.getField() + ": " + f.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return Result.fail(400, msg);
    }

    @ExceptionHandler(Exception.class)
    public Result<Void> handleOther(Exception e) {
        // ⚠️ 记录完整堆栈，但只给前端通用文案（不泄露内部信息）
        log.error("unexpected error", e);
        return Result.fail(500, "服务内部错误");
    }
}
```

⭐ **好处**：Controller 里不用写 try-catch，所有异常统一出口，前端永远收到同一种结构。

### 参数校验

```java
public record CreateOrderRequest(
    @NotNull(message = "userId 不能为空")
    Long userId,

    @Min(value = 1, message = "数量至少为 1")
    Integer quantity,

    @NotBlank(message = "收货地址不能为空")
    @Size(max = 200, message = "地址过长")
    String address
) {}
```

⚠️ Boot 3+ 用 **`jakarta.validation`**（不是 `javax.validation`）；需要依赖 `spring-boot-starter-validation`。

## 四、数据访问：三种常见选择

| 方案 | 特点 | 适合 |
|------|------|------|
| **JdbcTemplate** | 手写 SQL，轻量 | 简单查询、想完全控制 SQL |
| **MyBatis** | SQL 写在 XML/注解里，与 Java 解耦 | 复杂 SQL、需要精细优化（国内主流） |
| **Spring Data JPA** | 面向对象，方法名生成查询 | 领域模型清晰、CRUD 为主 |

⭐ 无论哪种，**索引与 SQL 优化的知识都通用**——数据库层的原理见 [[mysql/README]]。

### 声明式事务

```java
@Service
public class OrderService {

    @Transactional(rollbackFor = Exception.class)
    public void create(CreateOrderRequest req) {
        Order order = orderRepository.save(req.toEntity());
        inventoryService.deduct(req.productId(), req.quantity());  // 同事务
        // 任何 RuntimeException/Exception 都会回滚
    }
}
```

⚠️ **四种失效场景**（机制细节见 [[java/02-中段阶段-专题精讲/SpringBoot/IoC与Bean生命周期]]）：内部调用、非 public、异常被吞、checked 异常未配 `rollbackFor`。

⚠️ **事务里不要做远程调用**（HTTP/RPC）——外部调用可能耗时数秒，事务一直持有数据库连接与行锁。正确顺序：**先做远程校验 → 再开事务写库**。

### 连接池（HikariCP，Boot 默认）

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20        # ⚠️ 不是越大越好，按 DB 承受能力定
      minimum-idle: 5
      connection-timeout: 3000     # 拿连接超时（毫秒）
      max-lifetime: 1800000        # 30 分钟，略小于 DB 的 wait_timeout
```

⚠️ `maximum-pool-size` 过大会把数据库连接打满——**单实例 10–20 是常见合理区间**，多实例要按总数算。

## 五、配置与环境隔离

```yaml
# application.yml
spring:
  application:
    name: order-service
  profiles:
    active: dev          # 默认环境

---
# application-dev.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/shop

---
# application-prod.yml
spring:
  datasource:
    url: jdbc:mysql://${DB_HOST}:3306/shop
    username: ${DB_USER}
    password: ${DB_PASSWORD}    # ⚠️ 密码走环境变量/密钥管理，不写进仓库
```

⭐ **配置优先级**（高 → 低，常用部分）：命令行参数 > 环境变量 > `application-{profile}.yml` > `application.yml`。

### 类型安全配置绑定

```java
@ConfigurationProperties(prefix = "app.payment")
public record PaymentProperties(String endpoint, int timeoutSeconds) {}
```

比到处 `@Value("${...}")` 更安全、可校验、IDE 可跳转。

## 六、线程池与虚拟线程（2026 现状）

### 传统线程池

```java
@Bean
public ThreadPoolExecutor bizExecutor() {
    return new ThreadPoolExecutor(
        8, 16,                       // 核心 / 最大线程数
        60, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(1000),   // ⚠️ 必须设容量，否则会 OOM
        new ThreadPoolExecutor.CallerRunsPolicy()   // 满了让调用方执行（背压）
    );
}
```

⚠️ **不要用 `Executors.newFixedThreadPool()`**——队列无界，任务堆积会 OOM（《阿里巴巴 Java 开发手册》明确禁止）。

### 虚拟线程（Boot 3.2+，一行开启）

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

⭐ 适用：**IO 密集**（大量等待数据库/HTTP）。不适用：CPU 密集（虚拟线程不会让计算变快）。

## 七、打包与部署

```bash
mvn clean package -DskipTests
java -jar target/order-service-1.0.0.jar --spring.profiles.active=prod
```

### 容器部署的两个必配参数

```dockerfile
FROM eclipse-temurin:25-jre
COPY target/order-service-1.0.0.jar /app.jar
# ⭐ 让 JVM 感知容器内存上限（不要写死 -Xmx）
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "/app.jar"]
```

| 参数 | 作用 |
|------|------|
| `-XX:MaxRAMPercentage=75` | 堆最多用容器内存的 75%（而不是宿主机内存） |
| `-XX:+UseContainerSupport` | 容器感知（JDK 10+ 默认开启） |

⚠️ **不配这两个参数**，JDK 可能按宿主机内存算堆，容器被 OOMKilled。

### 可观测性（Actuator）

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when_authorized
```

| 端点 | 用途 |
|------|------|
| `/actuator/health` | 健康检查（K8s 探针用它） |
| `/actuator/metrics` | JVM、HTTP、连接池指标 |
| `/actuator/prometheus` | Prometheus 抓取格式 |

⭐ 配合 JVM 排查工具（[[java/02-中段阶段-专题精讲/JVM/排查工具与故障案例]]）形成完整监控闭环。

## 八、陷阱

1. **⚠️ Controller 里写业务逻辑**——接口层只做协议转换，业务进 Service；
2. **⚠️ Entity 直接当 DTO/VO**——表结构泄露、演进困难（加字段就影响接口）；
3. **⚠️ 事务里做远程调用**——连接与锁被长时间占用（这是线上最常见的吞吐杀手）；
4. **⚠️ 用 `Executors.newFixedThreadPool`**——无界队列会 OOM；
5. **⚠️ 容器里写死 `-Xmx`**——换规格就出事，用 `MaxRAMPercentage`；
6. **⚠️ 全局异常处理器把堆栈返回给前端**——信息泄露，同时前端也无法处理。

## 小结

- 三层边界：**Controller 做协议、Service 管业务与事务、Repository 只做数据访问**
- 三对象分离：**DTO 入、VO 出、Entity 只对数据库**
- 统一响应 + `@RestControllerAdvice` 全局异常是工程标配
- 事务在 Service 层，**不要包远程调用**；连接池按数据库承受能力配置
- 容器部署必配 **`-XX:MaxRAMPercentage`**，否则 OOMKilled
- Actuator 提供健康检查与指标，与 JVM 排查工具形成闭环

## 练习

1. 画出三层结构，标出 DTO / VO / Entity 各在哪一层边界上转换。
2. 写一个全局异常处理器，分别处理业务异常、参数校验异常、未知异常。
3. 为什么事务方法里不能调 HTTP 接口？正确顺序是什么？
4. `maximum-pool-size` 设成 100 有什么风险？
5. 容器里不配 `MaxRAMPercentage` 会发生什么？为什么？

## 导航

[[java/02-中段阶段-专题精讲/00-专题导航|返回专题导航]] · 上一篇：[[java/02-中段阶段-专题精讲/SpringBoot/IoC与Bean生命周期]] · 返回：[[java/README|Java 首页]]
