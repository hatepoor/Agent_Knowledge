---
tags: [interview, java, Spring, 工程实践, 面试, 题库]
type: guide
created: 2026-10-22
updated: 2026-10-22
---

# Spring 与工程实践

> **什么时候该读**：面试最后几轮——面试官不再问「语法会不会」，而是问「你写的 Spring 项目线上出过什么问题、怎么排查的」。这一章考的是**工程判断力**。
> **答题主线**：**「容器怎么造 Bean → 切面怎么织入 → 事务怎么失效 → 出问题怎么定位」**。四条线串起来，就是一个后端工程师的完整工作闭环。机制详解见 [[java/02-中段阶段-专题精讲/SpringBoot/IoC与Bean生命周期]] 与 [[java/02-中段阶段-专题精讲/SpringBoot/启动原理与自动配置]]。

---

## Q1. 什么是 IoC 和 DI？三种依赖注入方式怎么选？

**答：**

**结论：IoC（Inversion of Control，控制反转）= 对象的创建权从「你」交给「容器」；DI（Dependency Injection，依赖注入）= IoC 的具体实现手法——容器把依赖「推」进对象。**

```java
// ❌ 没有 IoC：自己 new，强耦合、无法替换、无法统一管理
public class OrderService {
    private UserRepository repo = new MySqlUserRepository();
}

// ✅ 有 IoC：容器 new 并注入
@Service
public class OrderService {
    private final UserRepository repo;

    public OrderService(UserRepository repo) {   // 依赖由容器注入
        this.repo = repo;
    }
}
```

**三种注入方式的取舍**：

| 方式 | 写法 | 优点 | 缺点 |
|------|------|------|------|
| ⭐ **构造器注入** | 构造器参数 | 依赖不可变（`final`）、**启动时就能发现循环依赖**、便于单测 | 参数多时略啰嗦 |
| **Setter 注入** | `@Autowired` 在 setter 上 | 可动态替换 | 依赖可变、可能 NPE |
| **字段注入** | `@Autowired` 在字段上 | 写法最短 | ⚠️ **隐藏依赖**、无法 `final`、**单测必须起容器** |

**⭐ 官方推荐构造器注入**——Spring 团队从 **4.3 起**明确建议，且**单构造器时可以省略 `@Autowired`**。

**⭐ 构造器注入的三个隐藏价值（面试加分点）**：

1. **依赖可以声明为 `final`**——对象一旦构造完成，依赖就不可变，**从根本上杜绝了「运行中被改」的问题**；
2. **循环依赖在启动时就暴露**——字段注入是「先造 A 再塞 B」，构造器注入是「必须先有 B 才能造 A」，**构造器循环依赖会直接启动失败并给出清晰报错**，而字段注入的循环依赖靠三级缓存兜着、问题被藏起来；
3. **单测不用起 Spring 容器**——`new OrderService(mockRepo)` 就行，测试快且不依赖框架。

**追问：为什么字段注入的单测必须起容器？**
→ 因为你**没法从外部传入依赖**——只有 Spring 反射能填那个字段。⚠️ 这也是为什么 Spring 4.3 之后提供了 `ObjectProvider<T>` 和构造器注入两个替代方案：前者让你**显式声明「这个依赖我可能要可能要」**（`getIfAvailable()`），后者让你**必须显式声明依赖**。

**追问：IoC 的代价是什么？**
→ **控制反转意味着你不知道对象什么时候被创建、注入了什么。** 具体表现：① 循环依赖；② **单例 Bean 的线程安全责任转移给开发者**（容器只管创建，不管并发安全）；③ **AOP 代理导致「对象不是它自己」**（`this` 调用绕过代理——这是事务失效的根源）。**IoC 不是纯粹的收益，它是「用可控性换灵活性」的交换。**

---

## Q2. Bean 的生命周期是什么？AOP 代理是在哪一步生成的？

**答：**

**结论：八步链路，AOP 代理在第 ⑥ 步（`postProcessAfterInitialization`）生成——这决定了「初始化方法里调自己的事务方法」事务不生效。**

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

**⭐ 两个必背的细节**：

| 细节 | 说明 |
|------|------|
| **AOP 代理在第 ⑥ 步** | 所以 **第 ⑤ 步（`@PostConstruct`）里调用自己的 `@Transactional` 方法，事务不生效**——此时容器里放的还是原始对象，代理还没织入 |
| **`@PostConstruct` 早于 `afterPropertiesSet`** | 两者都早于「代理对象可用」 |

**关于 `BeanPostProcessor` 的一个重要认知**：

`BeanPostProcessor` 是**Spring 的核心扩展点**——它不是「给业务 Bean 用的后置处理器」，而是**Spring 自己的基础设施**。AOP 代理、`@Async` 异步、`@Scheduled` 定时任务、`@Cacheable` 缓存，**全是靠不同的 `BeanPostProcessor` 织入的**。⚠️ **所以 `BeanPostProcessor` 本身会被提前实例化**（这会影响它的依赖注入顺序和 `@ComponentScan` 的时序——是 Spring 的经典「时序坑」来源）。

**追问：`InitializingBean` 和 `@PostConstruct` 有什么区别？**
→ 执行顺序：`@PostConstruct` → `InitializingBean.afterPropertiesSet()` → 自定义 `init-method`。**`@PostConstruct` 更通用**（不要求实现接口、JSR-250 标准）；**`InitializingBean` 更「Spring 原生」**（能在同一个类里看到逻辑，不用跳到注解上）。⚠️ **顺序不要依赖**——两个都写来做同样的初始化会重复执行。

**追问：Bean 的销毁怎么触发？**
→ ① 容器正常关闭（`SpringApplication` 结束时触发 `ContextClosedEvent`）；② 容器被 refresh 重建。⚠️ **但 `prototype` 作用域的 Bean，容器不负责销毁**——需要自己用 `@PreDestroy` 或 `DisposableBean` 处理，或借助 `DestructionAwareBeanPostProcessor`。

**追问：单例 Bean 能不能放成员变量？**
→ **能放不可变的（`final`），绝对不能放可变的。** 因为默认作用域是 `singleton`——**整个容器一个实例，所有请求共享**。⚠️ **往单例 Bean 的字段里塞「当前用户」「当前请求」是并发 Bug 的经典源头**。需要「每请求一份」的数据，用**方法级作用域的 Bean**、请求上下文（`RequestContextHolder`）或 **Scoped Values（Java 25）**。

---

## Q3. Spring 怎么解决循环依赖？为什么构造器循环依赖解决不了？

**答：**

**结论：靠三级缓存。构造器循环依赖**无解**——因为对象还没实例化，压根没法提供「半成品」。**

**三级缓存分别存什么**：

| 缓存 | 名称 | 存什么 |
|------|------|--------|
| **一级** | `singletonObjects` | 完整的成品 Bean |
| **二级** | `earlySingletonObjects` | 提前暴露的半成品（已实例化、未注入完） |
| **三级** | `singletonFactories` | 生成半成品的 `ObjectFactory`（⭐ **能在这里插入 AOP 代理**） |

**流程（A 需要 B，B 需要 A）**：

```
创建 A → 实例化后把 A 的 ObjectFactory 放入三级缓存
   ↓ 注入 B
创建 B → 注入 A 时，从三级缓存拿到 A 的工厂 → 生成早期引用 → 放入二级缓存
   ↓
B 创建完成 → 放入一级缓存
   ↓
A 注入 B 完成 → A 也放入一级缓存
```

**⭐ 为什么需要三级而不是两级？**

三级缓存的**工厂**让「**是否需要 AOP 代理**」这个判断**推迟到「真的被循环引用时」才做**——保证暴露给 B 的是**代理对象**而不是原始对象。如果只有两级（直接存半成品对象），A 被创建时还不知道它最终要不要被代理，暴露给 B 的可能是不带事务的原始对象，**事务就失效了**。

**两种解决不了的情况**：

| 情况 | 为什么无解 |
|------|---------|
| **构造器循环依赖** | ⚠️ **对象还没实例化就要依赖对方**——没有任何「半成品」可以暴露 |
| **prototype 作用域的循环依赖** | prototype 不进缓存，容器不负责创建和管理，拿不到早期引用 |

**⭐ 正确的解法不是「想办法让三级缓存兜住」，而是「改设计」**：

```java
// ❌ 构造器循环依赖 → 启动直接失败
@Service
class A { A(B b) {} }
@Service
class B { B(A a) {} }
```

| 解法 | 做法 |
|------|------|
| **① 改用 Setter/字段注入** | 能被三级缓存兜住，但**设计上仍是坏味道** |
| **② 引入第三个对象** | A 和 B 都依赖 C，C 不依赖它们——**最推荐** |
| **③ 事件解耦** | A 发事件，B 监听——**最彻底的解耦** |
| **④ `@Lazy` 注入** | `A(@Lazy B b)`——注入的是**代理**，真正调用时才解析依赖，打破循环 |

**追问：为什么构造器注入更容易暴露循环依赖，这是好事还是坏事？**
→ **是好事**。字段注入的循环依赖会被三级缓存悄悄兜住，**问题延后到运行时的某个诡异状态才爆出来**；构造器注入让它**在启动时就失败并给出清晰的 Bean 名称链**，排查成本低得多。**「启动就报错」优于「运行时偶发」。**

**追问：Spring Boot 2.6+ 默认禁止循环依赖了吗？**
→ ⚠️ **是的**——`spring.main.allow-circular-references` 在 **Spring Boot 2.6 起默认改为 `false`**（2.5.2 起引入该开关，2.6 默认关闭）。这是官方在**主动推动你解决循环依赖**，而不是依赖三级缓存兜底。⚠️ 遇到老项目升级后报 `BeanCurrentlyInCreationException`，**不要打开这个开关了事，要去改设计。**

---

## Q4. Spring AOP 的原理是什么？JDK 动态代理和 CGLIB 有什么区别？

**答：**

**结论：AOP 的底层是「代理」——Spring 在容器里放的其实不是你的原始对象，而是一个「长得像它、能拦截它的」代理对象。**

**三个核心概念**：

| 概念 | 含义 | 例子 |
|------|------|------|
| **切面（Aspect）** | 横切逻辑的模块 | 「事务切面」「日志切面」 |
| **切点（Pointcut）** | 在哪里切 | `@Transactional` 的匹配规则 |
| **通知（Advice）** | 切的时候做什么 | 前置/后置/**环绕**/异常 |

**两种代理实现**：

| | **JDK 动态代理** | **CGLIB** |
|---|---------------|---------|
| **原理** | 基于**接口**（`java.lang.reflect.Proxy`） | 基于**继承**（生成目标类的子类） |
| **要求** | 目标类必须实现接口 | 目标类不能是 `final` |
| **代理对象类型** | 代理的是接口，⚠️ **不能强转回实现类** | 代理的是目标类的子类，**可以强转** |
| **拦截范围** | **只拦接口方法** | **能拦类里的所有方法**（包括 `protected`、非接口方法） |

**⭐ Spring Boot 的代理默认值（版本敏感，必须说准）**：

| 版本 | `spring.aop.proxy-target-class` 默认 | 效果 |
|------|------------------------------------|------|
| **Spring Boot 2.x** | `true` | 默认 **CGLIB** |
| **Spring Boot 3.x** | `true` | 默认 **CGLIB** |
| **Spring Boot 4.x（当前线）** | `true` | 默认 **CGLIB** |

⚠️ **常见误解：「Boot 3 改回 JDK 代理了」——这是错的**。**Boot 2/3/4 都默认 CGLIB**（官方文档原话：「By default, Spring Boot's auto-configuration configures Spring AOP to use CGLib proxies」）。要换成 JDK 代理得显式配：

```yaml
spring:
  aop:
    proxy-target-class: false
```

⚠️ 注意区分**框架层**和**Boot 层**：`@EnableAspectJAutoProxy` 的 `proxyTargetClass` 属性默认是 `false`（框架层默认），但 **Spring Boot 的自动配置把它覆盖成了 `true`**——所以在 Boot 项目里实际生效的是 CGLIB。

**⭐ 三个必须知道的「代理陷阱」**：

| 陷阱 | 现象 | 原因 |
|------|------|------|
| **同类内部调用** | `this.other()` 上的 `@Transactional` 不生效 | `this` 是**原始对象**，不是代理，**不经过拦截器** |
| **`final` 类/方法** | CGLIB 代理不了 → **AOP 静默失效** | CGLIB 靠继承，`final` 没法重写 |
| **`private` 方法** | 任何代理都拦不到 | 代理只能拦「可被重写/可被接口声明」的方法 |

**追问：Spring 怎么在同类内部调用时让事务生效？**
→ 三种方案：

| 方案 | 做法 | 评价 |
|------|------|------|
| **① 拆分 Bean**（推荐） | 把 `inner()` 挪到另一个 Bean | ⭐ **最干净**，本质上是「拆职责」 |
| **② 注入自身代理** | `@Autowired @Lazy private OrderService self;` 然后 `self.inner()` | 可行但「自己注入自己」有点绕 |
| **③ `AopContext.currentProxy()`** | 需开启 `@EnableAspectJAutoProxy(exposeProxy = true)` | 能用但需要额外配置 |

**追问：`@Transactional` 用在 `private` 方法上会怎样？**
→ **静默失效**（不报错，只是事务不生效）。因为 CGLIB 靠子类重写方法，**`private` 方法没法被重写**；JDK 代理连 `private` 方法都看不到。**这是最容易被忽略的「为什么我的事务没生效」。**

**追问：AOP 的性能开销大吗？**
→ **有，但可控**。开销来自：① 每次调用多一层代理转发；② 代理对象本身占内存。**JDK 代理的实现类在启动时一次性生成（缓存），不是每次调用都生成**。⚠️ **但如果切点表达式写得太宽（比如 `execution(* com.example..*.*(..))`），会把所有方法都代理一遍**，这时才需要收敛切点。

---

## Q5. `@Transactional` 在什么情况下会失效？完整原因清单

**答：**

**结论：四种经典失效场景 + 两种容易忽略的场景。核心原因只有一个——「调用没有经过代理对象」。**

| # | 场景 | 机制 | 修复 |
|---|------|------|------|
| **1** | **同类内部调用**（`this.inner()`） | `this` 是原始对象，**绕过代理** | 拆到另一个 Bean / 注入自身代理 / 用 `TransactionTemplate` |
| **2** | **方法可见性不够** | ⚠️ **Spring 6.0 起**，基于类的（CGLIB）代理**默认也支持 `protected` 和包级私有方法**；但**基于接口的（JDK）代理仍只支持 `public`**。`private` 任何代理都拦不到 | 统一用 `public`（最稳）；或确保走 CGLIB 代理 |
| **3** | **异常被吞掉** | 切面收不到异常 → **感知不到要回滚** | 不要吞；或手动 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()` |
| **4** | **抛的是受检异常但没配 `rollbackFor`** | 默认只对 `RuntimeException` 和 `Error` 回滚 | `@Transactional(rollbackFor = Exception.class)` |
| **5** | **多线程里调用** | 事务绑定在**当前线程**的 `ThreadLocal` 上 | 在同一个线程里执行完整的事务方法；异步用 `@Async` 的要另想办法 |
| **6** | **类没被 Spring 管理**（自己 `new` 出来的） | 没有代理对象 | 交给容器管理 |

```java
@Service
public class OrderService {

    public void outer() {
        this.inner();          // ⚠️ 原因 1：同类内部调用，不走代理
    }

    @Transactional
    private void privateMethod() { }   // ⚠️ 原因 2：private，代理拦不到

    @Transactional
    public void swallow() {
        try {
            int x = 1 / 0;
        } catch (Exception e) {
            // ⚠️ 原因 3：异常被吞，事务感知不到 → 不回滚
        }
    }

    @Transactional
    public void checked() throws IOException {
        throw new IOException();    // ⚠️ 原因 4：默认只回滚 RuntimeException
    }
}
```

⚠️ **`private` 方法是「任何代理都拦不到」的那一类**——CGLIB 靠子类重写，`private` 方法不会被继承；JDK 代理更看不到它。**`protected` 和包级私有方法则要看代理类型**（Spring 6.0 起 CGLIB 默认支持，JDK 代理仍只支持 `public`）。**结论：事务方法统一写 `public`，最省心。**

**⭐ 面试官最爱追问：「为什么异常被吞了就不回滚？」**

**因为事务的回滚不是「数据库发现没有提交就自动回滚」，而是「切面在捕获到异常后主动调用 `rollback()`」。** 异常在业务代码里被 catch 了，切面就永远看不到它 → 切面按「正常返回」处理 → 提交。⚠️ **所以「吞异常」在事务里是一个真实的 bug，不是风格问题。**

**⭐ 回滚规则的完整表述**：

| 异常类型 | 默认行为 |
|---------|---------|
| `RuntimeException`（如 NPE） | ✅ **回滚** |
| `Error` | ✅ **回滚** |
| **受检异常**（如 `IOException`） | ❌ **不回滚**（提交） |

```java
@Transactional(rollbackFor = Exception.class)   // ⭐ 受检异常也回滚（推荐都加上）
@Transactional(noRollbackFor = BizException.class)  // 特定异常不回滚
```

**追问：`@Transactional` 的传播行为（Propagation）有哪些？**
→ 7 种，最常用的是：

| 传播行为 | 行为 |
|---------|------|
| **REQUIRED**（默认） | 有事务就加入，没有就新建 |
| **REQUIRES_NEW** | ⭐ **总是新建事务**，挂起当前事务——用于「无论主流程成败都要记日志」 |
| **SUPPORTS** | 有就加入，没有就以非事务方式执行 |
| **NESTED** | 在当前事务里开**嵌套事务**（用保存点，需 JDBC 支持） |
| **NOT_SUPPORTED** | **挂起**当前事务，以非事务方式执行 |
| **MANDATORY** | 必须在事务中，否则抛异常 |
| **NEVER` | 必须不在事务中，否则抛异常 |

⚠️ **`REQUIRES_NEW` 的坑**：它要**挂起当前事务并新开一个连接**——**连接池会被多占一份**，高并发下容易耗尽连接；而且被挂起的外层事务持有的行锁仍然存在，**容易和内层事务产生死锁**。

**追问：事务里能调远程接口吗？**
→ ⚠️ **不建议**。远程调用可能耗时数秒，事务会一直持有**数据库连接和行锁**——**这是线上最常见的吞吐杀手**。**正确顺序：先做远程校验 → 再开事务写库。**

---

## Q6. Spring Boot 的自动配置是怎么实现的？为什么加个 starter 就「什么都有了」？

**答：**

**结论：四步链路——注解 → 读注册文件 → 得到候选类 → 用条件注解筛选。**

**四步链路**：

```
① @EnableAutoConfiguration
      ↓ @Import(AutoConfigurationImportSelector.class)
② AutoConfigurationImportSelector
      读取 classpath 下所有 jar 中的注册文件
      ⚠️ Boot 2.7+：META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
         旧机制（Boot 2.7 前）：META-INF/spring.factories
      ↓
③ 得到候选配置类列表（几百个）
      ↓ 用条件注解逐个筛选
④ @ConditionalOnClass / @ConditionalOnMissingBean / @ConditionalOnProperty
      ↓ 满足条件的才注册为 Bean 定义
```

**三个最重要的条件注解**：

| 注解 | 含义 | 典型场景 |
|------|------|---------|
| `@ConditionalOnClass` | classpath 里有某个类才生效 | 引入了 `spring-boot-starter-web` 才装配 Web 相关 |
| `@ConditionalOnMissingBean` | ⭐ **容器里没有这个 Bean 才生效** | 你自己定义了 `DataSource`，Boot 就不自动创建 |
| `@ConditionalOnProperty` | 配置项满足条件才生效 | `spring.datasource.url` 存在才装配数据源 |

**⭐ `@ConditionalOnMissingBean` 是「可覆盖」的机制**——**自动配置永远只给「默认实现」，你一自定义，默认的就退让**。这是 Boot 能既「开箱即用」又「完全可接管」的根本原因。

**「加个 starter 就什么都有了」的完整链路**：

```
引入 spring-boot-starter-web
  → 它打包了 spring-webmvc、tomcat、jackson 等依赖
  → classpath 上出现这些类
  → @ConditionalOnClass 条件满足
  → WebMvcAutoConfiguration、JacksonAutoConfiguration 自动生效
  → 不用写一行配置就有了 REST 能力和 JSON 序列化
```

**⭐ 排查「为什么装配了 / 没装配」的三种手段**：

| 手段 | 做法 | 评价 |
|------|------|------|
| **`--debug`** | `java -jar app.jar --debug` | ⭐ **最快**——输出 **Condition Evaluation Report**，分 Positive matches（装配了，含原因）和 Negative matches（没装配，含哪个条件没过） |
| **Actuator** | 访问 `/actuator/conditions` | 线上用，JSON 版报告 |
| **手动排除** | `@SpringBootApplication(exclude = XxxAutoConfiguration.class)` | ⚠️ **排除后如果别处还依赖它，会报 `NoSuchBeanDefinitionException`——排除要成对处理依赖** |

**追问：`@SpringBootApplication` 是什么？**
→ 它是三个注解的合体：

```java
@SpringBootApplication
// 等价于：
@SpringBootConfiguration      // 标记为配置类（本质是 @Configuration）
@EnableAutoConfiguration      // ⭐ 开启自动配置
@ComponentScan                // 扫描当前包及子包的 @Component
```

⚠️ **`@ComponentScan` 的默认范围是「启动类所在包及其子包」**——**启动类必须放在根包下**（如 `com.example`），放在深层子包里上层 Bean 会扫不到。**这是 Spring Boot 最常见的「Bean 注入为 null」原因。**

**追问：`@SpringBootApplication` 和 `@ComponentScan` 有什么区别？**
→ **`@SpringBootApplication` 内部自带 `@ComponentScan`，且它是「就近扫描」**——**只扫描启动类自己所在包**。而单独写 `@ComponentScan(basePackages = "com.other")` 可以扫描任意包。⚠️ **两者同时存在时，`@ComponentScan` 会覆盖 `@SpringBootApplication` 里的扫描配置**（不是叠加，是覆盖）。

---

## Q7. Spring 常用注解有哪些？`@Autowired` 和 `@Resource` 有什么区别？

**答：**

**结论：注解按职责分四类记——「标组件」「装配」「切面」「配置」。**

**① 标注组件（让类能被扫描到）**：

| 注解 | 语义 | 本质 |
|------|------|------|
| `@Component` | 通用组件 | 基础注解，其余都是它的语义化特化 |
| `@Service` | 业务层 | 本质同 `@Component` |
| `@Repository` | 数据访问层 | ⚠️ **多了一个功能**：把持久层异常转成 Spring 的 `DataAccessException` |
| `@Controller` | 控制层 | `@Component` + `@ResponseBody` 语义 |
| `@RestController` | REST 控制层 | = `@Controller` + `@ResponseBody` |
| `@Configuration` | 配置类 | ⚠️ **多了一个功能**：类里的 `@Bean` 方法会被**CGLIB 增强**（代理模式），保证单例语义 |

**⭐ `@Configuration` 的「代理增强」是高频考点**：

```java
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        return new DataSource(...);
    }

    @Bean
    public UserService userService(DataSource ds) {
        return new UserService(ds);
    }
}
```

⚠️ **如果没有 `@Configuration`（只是 `@Component`），容器对 `dataSource()` 的多次调用会创建多个 `DataSource` 实例**——这会让「`userService` 拿到的 `ds`」和「容器里的 `ds`」不是同一个对象。**`@Configuration` 的 CGLIB 代理保证了「无论调多少次，返回的都是同一个单例 Bean」。** 所以**配置类要用 `@Configuration`，不要用 `@Component`。**

**② 装配相关**：

| 注解 | 用途 |
|------|------|
| `@Bean` | 在配置类里声明一个 Bean（装配第三方类） |
| `@Autowired` | 按**类型**注入（多个候选时按名称消歧） |
| `@Resource` | ⚠️ **优先按名称，再按类型**（`jakarta.annotation.Resource`，不是 Spring 的） |
| `@Value` | 注入配置项（`@Value("${app.name}")`） |
| `@ConfigurationProperties` | ⭐ **类型安全**地绑定一组配置 |
| `@Qualifier` | 指定注入哪个候选 Bean |
| `@Primary` | 多个同类型候选时，指定默认的那个 |

**⭐ `@Autowired` vs `@Resource`**：

| | `@Autowired` | `@Resource` |
|---|-------------|------------|
| **来源** | Spring 独有 | **JDK/Jakarta 标准**（`javax.annotation` → **Boot 3+ 改成 `jakarta.annotation`**） |
| **匹配顺序** | **先按类型**，多个候选再按名称 | **先按名称（byName）**，找不到再按类型（byType） |
| **required** | 默认 `true`（找不到就启动失败），可设 `false` | **默认 `true`**，但 `required` 属性行为不统一 |
| **支持构造器** | ✅ | ❌ **只支持字段和 setter** |

**结论：优先用 `@Autowired` 或构造器注入**——`@Resource` 的「先按名称」行为经常造成意外（字段名和 Bean 名不一致时行为反直觉），而且它是 Jakarta 规范的（Boot 3+ 要用 `jakarta.annotation.Resource`）。

**③ AOP / 切面相关**：

`@Aspect`（声明切面）、`@Before` / `@After` / `@AfterReturning` / `@AfterThrowing` / `@Around`（五种通知）、`@Pointcut`（切点表达式）。

**④ 配置与启动相关**：

| 注解 | 作用 |
|------|------|
| `@SpringBootApplication` | ⭐ 启动类三大功能合一 |
| `@EnableAutoConfiguration` | 开启自动配置 |
| `@Configuration` / `@Bean` | 配置类与 Bean 声明 |
| `@ControllerAdvice` + `@ExceptionHandler` | ⭐ 全局异常处理 |
| `@RestControllerAdvice` | REST 场景的全局异常处理 |
| `@Valid` / `@Validated` | 触发参数校验（Boot 3+ 用 `jakarta.validation`） |
| `@Transactional` | 声明式事务 |
| `@Async` / `@Scheduled` / `@Cacheable` | 异步 / 定时 / 缓存 |

**追问：`@ConfigurationProperties` 比 `@Value` 好在哪？**
→ | | `@Value` | `@ConfigurationProperties` |
|---|-----------|--------------------------|
| **类型安全** | ⚠️ 全靠字符串，编译期不检查 | ✅ 强类型绑定，**配置写错启动就报错** |
| **批量绑定** | 一个字段一个注解 | ✅ 一组配置绑到一个对象 |
| **IDE 支持** | ⚠️ 无 | ✅ 能跳转、有元数据 |
| **校验** | 需 `@Validated` 配合 | ✅ 直接加 `@Min`/`@NotBlank` 等 |

```java
@ConfigurationProperties(prefix = "app.payment")
public record PaymentProperties(String endpoint, int timeoutSeconds) {}
```

**追问：`@Repository` 注解的额外功能是什么？**
→ Spring 的**异常转换**（PersistenceExceptionTranslationPostProcessor）——把数据访问层的原生异常（`SQLException` 等）**转换成 Spring 的 `DataAccessException` 体系**（`DataIntegrityViolationException` 等），让上层不用处理具体数据库的异常。⚠️ **这个转换只对通过 `@Repository` 注解声明的 Bean 生效**——手写 DAO 类要自己继承 `JdbcDaoSupport` 或做转换。

---

## Q8. Spring Boot 启动的完整过程是什么？`refresh()` 里发生了什么？

**答：**

**结论：启动七步，核心是 `refresh()` 的 12 个子步骤——其中三步决定了「Bean 从哪来、端口什么时候开、对象什么时候创建」。**

**启动七步**：

```
① SpringApplication 构造
   推断应用类型（Servlet / Reactive / None）、加载 Initializer 与 Listener

② run() 开始
   启动计时器、发布 ApplicationStartingEvent

③ 准备 Environment
   读 application.yml / 环境变量 / 命令行参数

④ 创建 ApplicationContext
   Servlet 应用 → AnnotationConfigServletWebServerApplicationContext

⑤ refresh()  ← 核心，12 个子步骤
⑥ 启动内嵌 Tomcat（第 ⑨ 步）
⑦ 发布 ApplicationReadyEvent
   ⭐ 到这里服务才真正可用
```

**`refresh()` 的 12 步里，三个关键点**：

| 步骤 | 做什么 | 为什么关键 |
|------|-------|-----------|
| **⑤ `invokeBeanFactoryPostProcessors`** | ⭐ **自动配置在这里被解析** | 决定了「容器里有哪些 Bean 定义」 |
| **⑨ `onRefresh`** | ⭐ **创建并启动内嵌 Tomcat** | 端口开始监听（⚠️ **此时还没接收业务请求**） |
| **⑪ `finishBeanFactoryInitialization`** | ⭐ **实例化所有非懒加载的单例 Bean** | 业务对象真正被创建 |

**⭐ 一个必须理解的细节**：

**「内嵌 Tomcat 在第 ⑨ 步启动了，但 Bean 在第 ⑪ 步才实例化」——那中间谁来处理请求？**

答：**第 ⑨ 步只创建并「启动」容器（绑定端口），但 Servlet 容器此时还不会真正分发业务请求**——真正的请求处理要等到 `finishRefresh()`（第 ⑫ 步）之后。所以：

- **端口能连上（TCP 握手成功）≠ 服务可用**；
- **真正的「服务就绪」信号是 `ApplicationReadyEvent`**；
- **K8s 的就绪探针（readinessProbe）应该等 `ApplicationReadyEvent`**，而不是等端口通。

**这个细节能解释一个线上现象**：**服务刚启动时端口已通，但请求进来会 503 或超时**——因为 Bean 还没初始化完（尤其是 `@PostConstruct` 里做了耗时初始化）。

**追问：Bean 的实例化顺序是怎么决定的？**
→ ① **按依赖关系**——有依赖的先创建（`finishBeanFactoryInitialization` 里递归解析依赖）；② **同层内按注册顺序**（大致是 `@ComponentScan` 的扫描顺序，⚠️ **不保证严格有序**）；③ ⚠️ **不要依赖 Bean 的创建顺序**——那是很脆弱的耦合。⭐ **需要顺序就用 `@DependsOn` 或 `@Order`**（后者管的是**执行顺序**，不是创建顺序）。

**追问：`ApplicationStartedEvent` 和 `ApplicationReadyEvent` 有什么区别？**
→ `ApplicationStartedEvent`：**上下文已刷新**，但**Runner 还没跑完**；`ApplicationReadyEvent`：**Runner 都跑完了，服务真正可用**。⚠️ **Runner 里做耗时初始化会延迟就绪**——所以 Runner 应该尽量轻，耗时任务交给独立的调度器。

**追问：Spring Boot 4.x 的版本要点？**
→ **Spring Framework 7**、**Jakarta EE 11**（`javax.*` → `jakarta.*`，⚠️ 升级的主要工作量在这）、**Jackson 3**、**Hibernate 7**、**Java 基线 17+（推荐 25）**。⚠️ **Spring Boot 4.1 只支持 Java 17–26——装了 JDK 27 反而会报错**，「装最新版」在这里是错的。

---

## Q9. 线上服务出问题了，你的排查思路是什么？

**答：**

**结论：先分诊（判断是哪类问题），再按固定路径走。不要一上来就改代码重启。**

**第一步：分诊——四类问题对应四个方向**：

| 现象 | 大概率方向 | 先做什么 |
|------|-----------|---------|
| **CPU 100%** | 死循环 / GC 太频繁 / 计算太重 | `top -Hp` + `jstack`（详见 [[java/interview/05-JVM内存与垃圾回收]]） |
| **内存持续上涨 / OOM** | 内存泄漏 / 无界缓存 / 一次加载太多 | `jstat -gcutil` 看老年代趋势 → 导堆快照 → MAT 看 Dominator Tree |
| **接口变慢但 CPU/内存正常** | ⭐ **外部依赖**（DB/下游/连接池） | `jstack` 看线程卡在哪 + `trace` 看耗时分布 |
| **大量 5xx** | 代码异常 / 依赖挂了 / 线程池耗尽 | 看日志的异常栈 + 看线程池指标 |

**⭐ 第三类是最容易被误判的**——**CPU 和内存都正常却变慢，90% 是外部依赖问题**。这时 JVM 工具的作用是「**证明它卡在哪一层**」，而不是「优化 JVM」。

**第二步：Spring Boot 场景的专属检查点**：

| 检查项 | 怎么查 | 常见问题 |
|--------|-------|---------|
| **数据库连接池** | Actuator `/actuator/metrics/hikaricp.connections.pending` | ⚠️ **pending 高 = 连接不够**，请求在排队等连接 |
| **Tomcat 线程池** | `/actuator/metrics/tomcat.threads.busy` | 线程全忙 = 慢请求把线程占住了（⚠️ **线程池耗尽 = 整个服务不可用**） |
| **业务线程池** | 自定义埋点（活跃数/队列长度/拒绝数） | ⚠️ **队列堆积 + 拒绝数上升 = 容量不足** |
| **连接池配置** | `maximum-pool-size` | ⚠️ **不是越大越好**——单实例 10~20 是常见合理区间，多实例要按总数算；**太大把数据库打满** |

**第三步：常用工具清单**：

| 工具 | 用途 |
|------|------|
| **Actuator** | `/actuator/health`、`/metrics`、`/prometheus`、`/conditions`（自动配置诊断） |
| **Arthas** | 线上不改代码排查：`thread -n`、`trace`、`watch`、`dashboard` |
| **GC 日志** | `-Xlog:gc*`，判断 GC 是否是瓶颈 |
| **APM（SkyWalking / Pinpoint / 阿里 ARMS）** | ⭐ **生产环境的首选**——有完整的调用链和火焰图，不用现场 attach |

**⭐ 排查的通用心法**：

1. **先看现象分类，再看证据**——不要凭直觉跳到「肯定是数据库慢」；
2. **一次只查一件事**——同时开十个方向只会更乱；
3. **优先用 APM 的链路数据**（有历史基线），再考虑 attach 工具；
4. **⚠️ 任何「重启一下就好了」的结论都是未完成的排查**——问题必然复发；
5. **每次故障后补一条监控/告警**——**没有监控的故障一定会复发第二次**。

**追问：线上能直接用 `jmap -dump` 吗？**
→ ⚠️ **不要在流量高峰随便用**——它会 **STW 数秒**。**优先用「OOM 自动 dump」（提前配 `-XX:+HeapDumpOnOutOfMemoryError`）或 Arthas 的 `heapdump`。**

**追问：怎么预防「连接池被打满」？**
→ ① **控制 `maximum-pool-size`**（按数据库承受能力，不是越大越好）；② **⚠️ 事务里不做远程调用**（长时间占用连接）；③ **给连接获取超时**（`connection-timeout`），让请求快速失败而不是无限排队；④ **用 `max-lifetime` 略小于数据库的 `wait_timeout`**（避免拿到已被数据库关闭的死连接）；⑤ **监控 pending 数量并告警**。

---

## Q10. Spring Boot 工程实践里，哪些约定能显著减少线上事故？

**答：**

**结论：分四类——「配置」「分层」「资源」「可观测」。每一条都对应一类真实事故。**

**① 配置类约定**：

| 约定 | 防什么事故 |
|------|-----------|
| **密码走环境变量 / 密钥管理，不进仓库** | 密钥泄露 |
| **按 profile 分离环境**（`application-dev.yml` / `-prod.yml`） | 带着测试配置连生产库 |
| **容器里用 `-XX:MaxRAMPercentage=75`，不写死 `-Xmx`** | ⚠️ **换规格就 OOMKilled**（JVM 按宿主机内存算堆） |
| **用 `@ConfigurationProperties` 而不是散落的 `@Value`** | 配置写错、无法统一管理 |
| **配置优先级：命令行 > 环境变量 > profile > 默认** | 知道「哪个覆盖了哪个」 |

**② 分层约定**：

| 约定 | 防什么事故 |
|------|-----------|
| **Controller 只做协议转换**（参数校验、DTO→VO） | 业务逻辑散落，无法测试 |
| **事务边界在 Service 层** | 事务范围过大、连接被长时间占用 |
| **Entity 不直接返回给前端** | ⚠️ **表结构泄露**、敏感字段泄露、无法独立演进 |
| **DTO / VO / Entity 三者分离** | 接口与数据模型耦合，改表就改接口 |

**③ 资源与并发约定**：

| 约定 | 防什么事故 |
|------|-----------|
| **不用 `Executors.newFixedThreadPool()`** | ⚠️ **无界队列 OOM**（阿里规约明令禁止） |
| **自建线程池必设有界队列 + 明确拒绝策略** | 任务堆积 OOM、过载无感知 |
| **拒绝策略要埋监控** | 过载了没人知道 |
| **不吞异常**（尤其在事务方法里） | ⚠️ **事务不回滚、数据不一致** |
| **用 try-with-resources** | 资源泄漏（连接、流） |
| **流式导出 / 分页查询** | ⚠️ **一次加载百万行 → OOM**（线上最经典的 OOM 原因） |

**④ 可观测性约定**：

| 约定 | 价值 |
|------|------|
| **上线前配好 `HeapDumpOnOutOfMemoryError` + GC 日志** | ⚠️ **没配这两样 = 出事时盲诊** |
| **暴露 Actuator 的 `health` / `metrics` / `prometheus`** | K8s 探针 + 监控告警的基础 |
| **业务埋点：请求量、RT、错误率、线程池队列长度** | 容量问题的早期信号 |
| **统一响应结构 + 全局异常处理** | 前端不用处理多种错误格式；⚠️ **但不要把堆栈返回给前端**（信息泄露） |

**⭐ 面试加分的一条**：

**「我们线上出问题时的标准流程是：先看监控（是流量涨了还是依赖挂了）→ 再看 APM 链路（慢在哪一跳）→ 必要时用 Arthas/GC 日志深入 → 恢复后补监控和复盘。」**

**这个回答比罗列工具名有价值得多**——它证明你**有一套稳定的排障流程，而不是靠运气**。

**追问：你觉得团队最容易出问题的地方是什么？**
→ 这是开放题，**给出有观点的回答比给正确答案更重要**。可参考的观察角度：
- **「代码能跑但没人敢改」**——测试覆盖不足、缺少监控，谁改都怕出事故；
- **「配置散落在各处」**——同一个超时时间在五个文件里各配一次，改了一处漏了三处；
- **「新人上手成本高」**——没有统一的分层约定，每个人一套写法。

**追问：怎么给团队定规范？**
→ **别靠文档，靠「默认行为 + 检查工具」**——比如用 ArchUnit 在编译期检查分层依赖、用 Checkstyle/Spotless 统一格式、用 Spring Boot 的 `application.yml` 结构约定统一配置位置。⚠️ **靠人 review 的规范一定会腐化，靠工具强制才不会。**

---

## 本篇自测

- [ ] IoC 与 DI 的定义，三种注入方式的取舍，构造器注入的三个隐藏价值
- [ ] Bean 生命周期八步，AOP 代理在第 ⑥ 步，两个必背细节
- [ ] 三级缓存各存什么，为什么需要三级，构造器循环依赖为何无解
- [ ] AOP 的切面/切点/通知，JDK 代理与 CGLIB 差异，三个代理陷阱
- [ ] `@Transactional` 六种失效场景，回滚规则的完整表述，传播行为与事务内远程调用
- [ ] 自动配置四步链路，三个条件注解的作用，`--debug` 的条件报告
- [ ] Spring 常用注解分类，`@Autowired` 与 `@Resource` 的匹配顺序差异，`@Configuration` 的 CGLIB 增强
- [ ] 启动七步与 `refresh()` 三关键点，「端口通 ≠ 服务可用」的原因
- [ ] 四类线上问题的分诊路径，连接池与线程池的专属检查点
- [ ] 工程实践四类约定（配置/分层/资源/可观测）与各自防范的事故

---

## 导航

[[java/interview/README|Java 面试题库首页]] · 上一篇：[[java/interview/06-并发编程]] · 下一篇：返回：[[java/interview/README|Java 面试题库首页]]
