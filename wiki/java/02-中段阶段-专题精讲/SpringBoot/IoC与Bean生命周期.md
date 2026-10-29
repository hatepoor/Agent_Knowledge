---
tags: [java, 中段, SpringBoot, IoC, Bean, AOP, 事务]
type: guide
created: 2026-10-21
updated: 2026-10-21
---

# IoC 与 Bean 生命周期

> **什么时候该读**：写 Spring Boot 项目却总感觉「Bean 从哪来的说不清」；或者遇到循环依赖、`@Transactional` 不生效、AOP 切不到。

## 学习目标

读完本篇，你应当能够：

1. 用一句话解释 IoC 与 DI，并说出三种注入方式的取舍
2. 完整背出 Bean 生命周期（含 AOP 代理的织入时机）
3. 解释三级缓存如何解决循环依赖，以及哪种循环依赖解决不了
4. 说清 `@Transactional` 失效的四种原因

## 前置知识

| 前置 | 为什么需要 |
|------|-----------|
| [[java/02-中段阶段-专题精讲/SpringBoot/启动原理与自动配置]] | Bean 是自动配置的产物 |

---

## 一、IoC 与 DI：把「new」交给容器

### 没有 IoC 的世界

```java
public class OrderService {
    // 自己 new：强耦合、无法替换、无法统一管理
    private UserRepository repo = new UserRepositoryImpl();
}
```

### 有 IoC 的世界

```java
@Service
public class OrderService {
    private final UserRepository repo;

    // 依赖由容器注入（构造器注入）
    public OrderService(UserRepository repo) {
        this.repo = repo;
    }
}
```

| 概念 | 含义 |
|------|------|
| **IoC（控制反转）** | 对象的创建权从「你」反转给「容器」 |
| **DI（依赖注入）** | IoC 的具体实现：容器把依赖「推」进对象 |

⭐ 一句话：**你不用 `new`，容器帮你 `new` 并塞好依赖。**

### 三种注入方式

| 方式 | 写法 | 优点 | 缺点 |
|------|------|------|------|
| ⭐ **构造器注入** | 构造器参数 | 依赖不可变（`final`）、**启动时就能发现循环依赖**、便于单测 | 参数多时略啰嗦 |
| Setter 注入 | `@Autowired` 在 setter 上 | 可动态替换 | 依赖可变，容易 NPE |
| 字段注入 | `@Autowired` 在字段上 | 写法最短 | ⚠️ 隐藏依赖、无法 `final`、**单测必须靠容器** |

⭐ **官方推荐构造器注入**——Spring 团队从 4.3 起明确建议；从 Spring 4.3 开始单构造器可省略 `@Autowired`。

## 二、Bean 是什么与作用域

**Bean = 由 Spring 容器创建和管理的对象。**

| 作用域 | 含义 | 注意 |
|--------|------|------|
| **singleton**（默认） | 整个容器一个实例 | ⚠️ 必须线程安全（无状态设计） |
| prototype | 每次获取都新建 | 容器不负责销毁 |
| request / session | Web 请求 / 会话级 | 仅 Web 环境 |

⚠️ **默认单例意味着：Spring 的 Service 里不要放可变成员变量**——那是并发 Bug 的源头。

## 三、Bean 生命周期（完整链路）

```
① 实例化（构造器）
      ↓
② 属性填充（依赖注入）
      ↓
③ Aware 回调（BeanNameAware / BeanFactoryAware / ApplicationContextAware）
      ↓
④ BeanPostProcessor.postProcessBeforeInitialization()
      ↓
⑤ 初始化：@PostConstruct → InitializingBean.afterPropertiesSet() → 自定义 init-method
      ↓
⑥ BeanPostProcessor.postProcessAfterInitialization()   ⭐ AOP 代理在这里生成
      ↓
⑦ 使用（业务调用）
      ↓
⑧ 销毁：@PreDestroy → DisposableBean.destroy() → 自定义 destroy-method
```

⭐ **面试必问的两点**：

1. **AOP 代理在第 ⑥ 步生成**——所以「初始化方法里调用自己的事务方法」时代理还没织入，事务不生效；
2. **`@PostConstruct` 早于 `afterPropertiesSet`**，两者都早于「代理对象可用」。

## 四、三种装配方式

### ① 注解扫描（最常用）

```java
@Component        // 通用组件
@Service          // 业务层（语义化，本质同 @Component）
@Repository       // 数据访问层（还能把数据库异常转成 Spring 异常）
@Controller       // 控制层
@Configuration    // 配置类（内部 @Bean 方法生效）
```

### ② `@Bean` 配置类（装配第三方类）

```java
@Configuration
public class AppConfig {
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();   // 第三方类不能加 @Component，就用 @Bean
    }
}
```

### ③ 条件装配

```java
@Bean
@ConditionalOnMissingBean
public CacheService cacheService() { ... }
```

## 五、循环依赖与三级缓存

### 什么是循环依赖

```java
@Service
class A { A(B b) {} }
@Service
class B { B(A a) {} }
```

A 要 B、B 要 A——按构造器注入会直接启动失败；**字段/Setter 注入**则可能被解决。

### 三级缓存

| 缓存 | 存什么 |
|------|--------|
| **一级：singletonObjects** | 完整的成品 Bean |
| **二级：earlySingletonObjects** | 提前暴露的半成品（已实例化、未注入完） |
| **三级：singletonFactories** | 生成半成品的 ObjectFactory（**能在这里插入 AOP 代理**） |

流程（A ↔ B）：

```
创建 A → 实例化后把 A 的 ObjectFactory 放入三级缓存
   ↓ 注入 B
创建 B → 注入 A 时，从三级缓存拿到 A 的工厂 → 生成早期引用 → 放入二级缓存
   ↓
B 创建完成 → 放入一级缓存
   ↓
A 注入 B 完成 → A 也放入一级缓存
```

⭐ **为什么需要三级而不是两级**：三级缓存的工厂让「是否需要 AOP 代理」这个判断**推迟到真正被循环引用时**——保证最终暴露给 B 的是代理对象而不是原始对象。

⚠️ **两种情况解决不了**：

1. **构造器循环依赖**（对象还没实例化就要依赖）；
2. **prototype 作用域的循环依赖**。

## 六、AOP：事务、日志、权限的实现基础

### 三个概念

| 概念 | 含义 |
|------|------|
| **切面（Aspect）** | 横切逻辑的模块（如「事务」） |
| **切点（Pointcut）** | 在哪里切（表达式匹配的方法） |
| **通知（Advice）** | 切的时候做什么（前置/后置/环绕/异常） |

```java
@Aspect
@Component
public class LogAspect {
    @Around("execution(* com.example.service.*.*(..))")
    public Object log(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } finally {
            System.out.println(pjp.getSignature() + " 耗时 " + (System.currentTimeMillis() - start) + "ms");
        }
    }
}
```

### 两种代理

| 代理 | 条件 | 特点 |
|------|------|------|
| **JDK 动态代理** | 目标类实现了接口 | 基于接口，生成实现同一接口的代理类 |
| **CGLIB** | 目标类没有接口（Boot 2.x 起默认 `proxyTargetClass=true`） | 基于继承，生成子类（⚠️ 无法代理 `final` 类/方法） |

⭐ **代理是理解「@Transactional 失效」的钥匙**：调用必须**经过代理对象**，切面才会执行。

## 七、`@Transactional` 失效的四种原因（高频）

```java
@Service
public class OrderService {
    public void outer() {
        this.inner();          // ⚠️ 原因 1：同类内部调用，不走代理
    }

    @Transactional
    public void inner() { ... }

    @Transactional
    void packageMethod() { ... }   // ⚠️ 原因 2：非 public 方法，代理无法拦截

    @Transactional
    public void swallow() {
        try {
            int x = 1 / 0;
        } catch (Exception e) {
            // ⚠️ 原因 3：异常被吞，事务感知不到
        }
    }

    @Transactional
    public void checked() throws IOException {
        throw new IOException();    // ⚠️ 原因 4：默认只回滚 RuntimeException
    }
}
```

| 原因 | 机制 | 修复 |
|------|------|------|
| 同类内部调用 | `this.inner()` 绕过代理 | 拆到另一个 Bean；或注入自身代理（`AopContext.currentProxy()`）；或用 `TransactionTemplate` |
| 非 public | 代理只能拦截 public 方法 | 改为 public |
| 异常被吞 | 切面收不到异常，不会回滚 | 不要吞异常，或手动 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` |
| checked 异常 | `rollbackFor` 默认只含 `RuntimeException` | `@Transactional(rollbackFor = Exception.class)` |

⭐ 还有两种容易忽略的：**多线程里调用**（事务绑定线程）、**类没有被 Spring 管理**（自己 `new` 出来的对象没有代理）。

## 八、陷阱

1. **⚠️ 在单例 Bean 里放可变状态**——所有请求共享，并发必炸；Service 应该无状态；
2. **⚠️ 用字段注入图省事**——隐藏依赖、无法 `final`、单测难；
3. **⚠️ 以为「循环依赖能解决」就无所谓**——能解决的是「字段/Setter」场景，设计上就该拆；
4. **⚠️ 在 `@PostConstruct` 里调事务方法**——此时代理未织入；
5. **⚠️ `final` 类/方法上想加切面**——CGLIB 继承不了，AOP 静默失效。

## 小结

- **IoC/DI**：对象创建权交给容器；⭐ **构造器注入**是首选
- Bean 生命周期：**实例化 → 注入 → Aware → 前置处理 → 初始化 → 后置处理（AOP 代理） → 使用 → 销毁**
- 三级缓存解决字段/Setter 的循环依赖；**构造器循环依赖解决不了**
- AOP 靠**代理**实现（JDK 动态代理 / CGLIB）——**调用必须走代理，切面才生效**
- `@Transactional` 四大失效：**内部调用、非 public、异常被吞、checked 异常**

## 练习

1. 默写 Bean 生命周期的 8 步，指出 AOP 代理在哪一步生成。
2. 为什么推荐构造器注入？它带来什么额外好处（提示：循环依赖）？
3. 三级缓存分别存什么？为什么两级不够？
4. 列出 `@Transactional` 失效的四种原因与对应修复。
5. JDK 动态代理和 CGLIB 各适用于什么目标类？为什么 `final` 方法切不到？

## 导航

[[java/02-中段阶段-专题精讲/00-专题导航|返回专题导航]] · 上一篇：[[java/02-中段阶段-专题精讲/SpringBoot/启动原理与自动配置]] · 下一篇：[[java/02-中段阶段-专题精讲/SpringBoot/Web开发与分层实践]]
