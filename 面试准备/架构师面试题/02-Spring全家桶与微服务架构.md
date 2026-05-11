## 二、Spring全家桶与微服务架构

---

### 1. Spring IoC容器的底层实现、Bean生命周期、循环依赖？

#### IoC 容器的本质

Spring IoC 容器 = `BeanDefinition` 注册表 + `BeanFactory` + `ApplicationContext` 的一系列 `refresh()` 后置处理。

**核心数据结构**：
```java
// DefaultListableBeanFactory
Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);  // 定义注册
Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);             // 一级缓存：完全体
Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);         // 二级缓存：早期半成品
Map<String, ObjectFactory<?>> singletonFactories = new ConcurrentHashMap<>(16);  // 三级缓存：ObjectFactory
Set<String> singletonsCurrentlyInCreation = Collections.newSetFromMap(new ConcurrentHashMap<>(16));
```

**容器启动的 13 步（refresh() 核心）**：

1. `prepareRefresh()` → 准备刷新（清空缓存、设置状态、校验环境配置）
2. `obtainFreshBeanFactory()` → 解析 XML/注解/配置类，`refreshBeanFactory` + 注册 BeanDefinition
3. `prepareBeanFactory(beanFactory)` → 注册标准组件（`ApplicationContextAwareProcessor`、Environment）
4. `postProcessBeanFactory(beanFactory)` → 子类扩展点（WAC 中注册 `ServletContextAwareProcessor`）
5. `invokeBeanFactoryPostProcessors()` → 执行 BeanFactoryPostProcessor（最关键一步！）
   - `ConfigurationClassPostProcessor` 解析 `@Configuration`、`@ComponentScan`、`@Import`、`@Bean`
   - 这一步把注解驱动的配置解析为 BeanDefinition 注册到容器
6. `registerBeanPostProcessors()` → 注册 BeanPostProcessor（只注册，不执行）
7. `initMessageSource()` → 国际化
8. `initApplicationEventMulticaster()` → 事件广播器
9. `onRefresh()` → 子类扩展（Spring Boot 嵌入 Web 服务器在这里）
10. `registerListeners()` → 注册事件监听器
11. `finishBeanFactoryInitialization()` → **创建所有非 lazy 单例 Bean**（最关键）
12. `finishRefresh()` → 发布 `ContextRefreshedEvent`
13. `resetCommonCaches()` → 清除反射缓存

#### Bean 的完整生命周期（15个阶段）

```java
// 1. 实例化（构造器反射调用）
constructor.newInstance(args);

// 2. 属性填充（@Autowired、@Value）
populateBean(beanName, mbd, instanceWrapper);

// 3-5. Aware 接口（按顺序）
((BeanNameAware) bean).setBeanName(beanName);
((BeanClassLoaderAware) bean).setBeanClassLoader(classLoader);
((BeanFactoryAware) bean).setBeanFactory(this);

// 6. BeanPostProcessor.postProcessBeforeInitialization()
//    注意：@PostConstruct 是在 InitDestroyAnnotationBeanPostProcessor 中处理的

// 7. @PostConstruct 方法

// 8. InitializingBean.afterPropertiesSet()

// 9. init-method

// 10. BeanPostProcessor.postProcessAfterInitialization()
//     AOP 代理对象就是在这里生成的！！！！
//     AbstractAutoProxyCreator.wrapIfNecessary() 会判断是否需要创建代理

// 11. 「Bean 就绪」

// 12-14. 容器关闭
// @PreDestroy → DisposableBean.destroy() → destroy-method
```

**关键认知**：AOP 代理对象是在 `postProcessAfterInitialization()` 产生的，所以依赖注入时注入的是**原始 Bean 的早期引用**，但最终对外暴露的是**代理对象**。这就是为什么 `@Transactional` 自调用失效——自调用（`this.method()`）绕过了代理对象。

#### 循环依赖三级缓存机制（核心）

**为什么需要三级缓存**：

一级缓存（singletonObjects）存完全体。但 A 依赖 B、B 依赖 A，A 创建过程中填充属性时需要注入 B，B 还没创建。B 创建的过程中需要注入 A，此时 A 还没填充完属性。

三级缓存 `singletonFactories` 存的是 `ObjectFactory`（lambda），它能按需生成早期引用。为什么需要 ObjectFactory 而不是直接存早期 Bean？**因为 AOP**——如果 A 需要生成代理，ObjectFactory 可以在需要的时候才调用 `getEarlyBeanReference()` 生成代理对象。

**完整流程（A ↔ B 循环依赖）**：
```
1. 创建 A 实例（反射）
2. A 的 ObjectFactory 放入三级缓存（lambda: getEarlyBeanReference(A)）
3. 填充 A 属性 → 需要 B
4. 创建 B 实例
5. B 的 ObjectFactory 放入三级缓存
6. 填充 B 属性 → 需要 A
7. 从三级缓存获取 A 的 ObjectFactory → 调用 getEarlyBeanReference(A)
   → 如果 A 需要 AOP 代理，返回到这里的是代理对象
   → 放入二级缓存（earlySingletonObjects）
8. B 完成创建 → 放入一级缓存
9. 回到 A，B 已就绪 → 填充 B 到 A
10. A 完成创建 → 放入一级缓存 → 清理二三级缓存
```

**为什么构造器循环依赖无法解决**：因为构造器调用发生在实例化阶段，此时 Bean 还没创建完，更没有 ObjectFactory 可以暴露。Spring 只能抛 `BeanCurrentlyInCreationException`。解决方案：`@Lazy` 延迟注入（注入的是代理对象，真正使用时才初始化）。

**为什么 prototype 循环依赖也无法解决**：因为 prototype 不缓存，每次获取都创建新实例。三级缓存只缓存单例。

---

### 2. Spring AOP 原理：JDK 动态代理 vs CGLIB，实际工作场景？

#### JDK 动态代理

本质是**接口代理**。`Proxy.newProxyInstance` 生成一个实现目标接口的代理类（`class com.sun.proxy.$Proxy0`），所有方法调用拦截到 `InvocationHandler.invoke()`。

JDK 动态代理的限制：
- 必须基于接口
- 代理对象 `instanceof` 目标接口，但**不是** `instanceof` 目标类
- 代理对象是一个全新的类，不继承目标类

实现原理：
```java
// $Proxy0 extends Proxy implements UserService {
//   所有方法都委托给 InvocationHandler.invoke：
//   public User findById(Long id) {
//       return (User) h.invoke(this, m3, new Object[]{id});
//   }
// }
```

#### CGLIB 代理

本质是**子类代理**。通过 ASM 字节码操纵框架动态生成目标类的子类，重写所有非 final 方法，插入拦截逻辑。

CGLIB 的限制：
- final 类/方法不能代理
- 构造器至少调用两次（CGLIB 实例化 + 目标类初始化）
- 生成的类名 `xxx$$EnhancerBySpringCGLIB$$xxx`
- 比 JDK 代理略慢（创建开销大），但运行效率相近

**Spring 的选择逻辑**：
- Spring Boot 1.x：默认 JDK 代理（如果有接口）
- Spring Boot 2.x：默认 CGLIB（`spring.aop.proxy-target-class=true`）
- 为什么改：CGLIB 不需要接口限制，且 Spring 已集成 CGLIB 依赖

#### Spring AOP 的架构

AOP 在 Spring 中的角色定位是「横切关注点的模块化」。核心组件链：

```
@Aspect → Advisor（AspectJMethodBeforeAdvice + Pointcut）
       → AbstractAutoProxyCreator（BeanPostProcessor）
       → postProcessAfterInitialization → wrapIfNecessary
       → ProxyFactory → JDK 或 CGLIB 代理
```

**AOP 链调用过程**：
```java
// MethodInvocation.proceed() 链
BeforeAdvice → AroundAdvice.proceed() → AfterReturningAdvice → AfterAdvice → 目标方法
```

**注意区分 Spring AOP 和 AspectJ**：Spring AOP 是运行时动态代理，只能拦截 Spring Bean 的方法调用；AspectJ 是编译期织入，可以拦截任何方法（甚至构造器、字段访问）。

#### 实际 AOP 运用

**1. @Transactional 事务（最重要的运用）**
- 实现：`TransactionInterceptor`（MethodInterceptor）
- 流程：方法调用被拦截 → `DataSourceTransactionManager.getTransaction()` → `conn.setAutoCommit(false)` → `invoke 目标方法` → `commit/rollback`
- 注意：事务传播行为由 `TransactionDefinition` 定义，REQUIRED 会在已有事务上挂接，REQUIRES_NEW 会挂起当前事务创建新事务

**2. 自定义 @OperationLog（日志记录）**
- 用注解标记需要记录日志的方法
- 切面在方法执行前后记录入参、返回值和耗时

**3. @Cacheable（缓存抽象）**
- `CacheInterceptor` 拦截方法调用
- 如果缓存命中直接返回缓存值（不执行方法）
- 如果未命中执行方法并把返回值存入缓存

---

### 3. 微服务从0到1落地的完整流程？

#### 拆分原则深入版

**DDD 战略设计的核心**：识别限界上下文（Bounded Context）和上下文映射（Context Map）。

**拆分信号（什么时候一定要拆）**：
1. 不同模块的数据库表被同一事务持有（耦合）
2. 一个表被多个团队修改（同一表多 Ownership）
3. 一个功能改动需要改 N 个模块
4. CI 构建时间 > 20 分钟（代码量太大）
5. 团队人数 > 6-8 人（沟通成本超过拆分成本）

**拆分策略**：
- **按业务能力拆分**：用户服务、订单服务、商品服务
- **按 DDD 限界上下文拆分**：采购上下文、销售上下文、库存上下文
- **按变化速率拆分**：稳定模块（基础服务）和频繁变动的模块（业务规则）分开
- **按数据量/访问热度拆分**：热点服务和冷数据服务分开

**拆分粒度判断标准**：
- 一个微服务 = 1-2 个 DevOps 团队的 Owner
- 代码量约 1-5 万行 Java
- 可以独立部署和发布

**基础设施清单**：
```
注册中心  Nacos（CP+AP模式）
配置中心  Nacos/Apollo
网关      Spring Cloud Gateway
熔断限流  Sentinel
RPC调用   OpenFeign + LoadBalancer
链路追踪  SkyWalking
指标监控  Prometheus + Grafana
日志      ELK（Elasticsearch + Logstash + Kibana）
```

---

### 4. 微服务框架的深度定制与扩展？

#### 自定义 Starter 的完整架构

一个标准 Starter 应该包含两个模块：`autoconfigure` + `starter`（spring-boot-starter 只放依赖 + 配置文件）。核心三要素：

**① 配置属性类（`@ConfigurationProperties`）**：
```java
@ConfigurationProperties(prefix = "my.log")
public class LogProperties {
    private boolean enabled = true;
    private Set<String> excludeUrls = new HashSet<>();  // 支持集合配置
    private Level level = Level.INFO;
}
```

**② 自动配置类**：
```java
@AutoConfiguration  // Spring Boot 2.7+ 新注解，替代 @Configuration
@ConditionalOnProperty(prefix = "my.log", name = "enabled", havingValue = "true", matchIfMissing = true)
@EnableConfigurationProperties(LogProperties.class)
public class LogAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean  // 允许用户自定义覆盖
    @ConditionalOnClass(LogAspect.class)
    public LogAspect logAspect(LogProperties properties) {
        return new LogAspect(properties);
    }
}
```

**③ 注册到 spring.factories / auto-configuration-imports**：
```
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.LogAutoConfiguration
```

**条件注解的完整层级**：
- `@ConditionalOnClass`：classpath 中有某个类（starter 的核心保障，依赖没引入不会报错）
- `@ConditionalOnMissingBean`：用户没有自定义同名 Bean
- `@ConditionalOnProperty`：配置文件开关
- `@ConditionalOnBean`：某个 Bean 存在才生效
- `@ConditionalOnWebApplication`：Web 环境才生效
- `@ConditionalOnExpression`：SpEL 表达式条件

#### 实际扩展过的组件

**1. Feign 拦截器——全链路 TraceId 透传**：
```java
public class TraceFeignInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        String traceId = MDC.get("traceId");
        if (traceId != null) {
            template.header("X-Trace-Id", traceId);
        }
    }
}
```
关键点：不只是透传 traceId，还要把线程的上下文信息（用户 ID、租户 ID、AB 实验标签）都放进 Header，否则服务链后段无法做路由判断。

**2. MyBatis 数据权限拦截器**：
不能只靠 SQL 拼接（有注入风险），而是用 MyBatis 的 `Interceptor` + `@Signature` 在 `Executor.query/update` 阶段拦截 `BoundSql`，追加权限过滤的 PreparedStatement 参数。这样可以防止注入，同时不影响 SQL 的结构化分析。

---

### 5. 50+微服务稳定运行与平滑演进？

#### 稳定性保障的技术细节

**灰度发布的流量染色实现**：
1. 注册中心打标签：新版实例在 Nacos metadata 中加 `version=v2`
2. Spring Cloud LoadBalancer 基于 `HintBasedServiceInstanceListSupplier` 和 `ReactorServiceInstanceLoadBalancer` 选择匹配的实例
3. 网关注入流量标识：通过自定义 `GatewayFilter` 根据 UserId hash / 白名单 / AB 实验标签注入 `X-Version` Header

**版本管理的终极策略**：
- API 版本放在 URL 路径中：`/api/v1/order` + `/api/v2/order`（最清晰，网关路由简单）
- 兼容性保证：新增字段不删旧字段，废弃字段添加 `@Deprecated` 并标注移除时间
- 数据库 Schema 演进的黄金法则：**只增不减不改**。删除字段分两步：先标记废弃 → 下个大版本真删

#### 服务降级的分级策略

```
L0（核心链路）：不可降级（下单、支付）
L1（重要链路）：主流程失败后走备用方案（短信验证 → 语音验证）
L2（可降级链路）：失败后给 fallback（推荐商品 → 热榜兜底）
L3（可关闭链路）：压力大时直接关（积分增长、操作日志详情）
```

**Sentinel 实现降级**：配置 `@SentinelResource(fallback = "fallbackMethod")`，降级后返回兜底数据。

---

### 6. Spring事务的传播机制和隔离级别深度解析？

#### 传播机制的行为表（完整版）

| 传播行为 | 当前有事务 | 当前无事务 | 典型场景 |
|---------|-----------|-----------|---------|
| REQUIRED | 用当前事务 | 新建事务 | 默认选择，99% 场景 |
| REQUIRES_NEW | 挂起当前，新建独立事务 | 新建事务 | 日志记录（独立提交，不要因为主事务回滚而丢失日志） |
| SUPPORTS | 用当前事务 | 非事务执行 | 查询方法 |
| NOT_SUPPORTED | 挂起当前，非事务执行 | 非事务执行 | 不需要事务的长查询 |
| MANDATORY | 用当前事务 | 抛异常 | 强制调用方必须已开启事务 |
| NEVER | 抛异常 | 非事务执行 | 明确禁止在事务中执行 |
| NESTED | 创建 Savepoint 嵌套 | 新建事务 | 部分回滚（只支持 JDBC） |

#### REQUIRES_NEW 的陷阱

```java
@Transactional  // 外层事务
public void createOrder() {
    orderMapper.insert(order);            // 插入订单
    logService.asyncLog("创建订单");      // 调用 REQUIRES_NEW
    // logService 插入日志后，外层如果抛异常
    throw new RuntimeException("业务异常");
}
```

外层回滚 → order 插入回滚，但 `asyncLog` 的日志已经提交！这是预期行为（日志应该保留），但如果业务逻辑错误地使用 REQUIRES_NEW 会导致数据不一致。

#### 事务隔离级别与并发问题对照

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | InnoDB 实现 |
|---------|------|-----------|------|------------|
| READ UNCOMMITTED | ✓ | ✓ | ✓ | 几乎不用 |
| READ COMMITTED | ✗ | ✓ | ✓ | 每次快照读重建 ReadView |
| REPEATABLE READ (默认) | ✗ | ✗ | 部分解决 | 一次事务一个 ReadView + 间隙锁 |
| SERIALIZABLE | ✗ | ✗ | ✗ | 所有 SELECT 隐式加 LOCK IN SHARE MODE |

**RR 隔离下为什么还有幻读**：InnoDB 的 RR 通过间隙锁解决了"当前读"的幻读。但 `SELECT`（快照读）不受间隙锁保护。如果你在一个事务里先快照读，别人插入了数据，你再 `SELECT ... FOR UPDATE`（当前读），会看到新插入的数据——这就是 RR 下的幻读。

#### 事务失效的 7 个场景（完整版）

1. **同类方法调用（最常见，占 90% 失效案例）**：AOP 代理在 Bean 的外层，`this.method()` 直接调内部方法 → 绕过代理
2. **非 public 方法**：CGLIB 重写非 public 有风险
3. **异常被 catch**：`catch(Exception e) { log.error(e); }` 没有 rethrow，事务管理器看不到异常 → 提交了
4. **checked 异常不回滚**：`@Transactional` 默认 `rollbackFor = {RuntimeException.class, Error.class}`
5. **数据库引擎**：MyISAM 不支持事务
6. **多线程**：事务通过 ThreadLocal 绑定，子线程不在同一事务中
7. **`@Transactional` 加在接口上**：JDK 动态代理不会读取接口注解

---

### 7. Spring Boot自动配置原理与自定义Starter？

#### 自动配置的完整加载链

```
@SpringBootApplication
  → @EnableAutoConfiguration
    → @Import(AutoConfigurationImportSelector)

AutoConfigurationImportSelector.selectImports():
  1. 读取 spring.factories / auto-configuration-imports
  2. 过滤 @Conditional 不满足的配置
  3. 排序 (@AutoConfigureOrder, @AutoConfigureBefore/After)
  4. 按顺序加载
```

**为什么有些 Starter 引入就能用，有些还要加配置**：以 `DataSourceAutoConfiguration` 为例，它用 `@ConditionalOnClass(DataSource.class)` 判断驱动在 classpath 中，用 `@ConditionalOnMissingBean(DataSource.class)` 判断用户没有自定义。如果你加了配置 → Spring Boot 创建你指定的；如果你没加配置 → 不存在 `DataSource.class` → 跳过了。

#### 条件注解源码级细节

**`@ConditionalOnMissingBean` 的判断时机**：在当前 Bean 定义注册之后，Spring 容器会检查是否有先前定义的 Bean。问题在于：如果两个自动配置类互相用 `@ConditionalOnMissingBean` 检查对方，可能导致加载顺序影响结果。`@AutoConfigureBefore/After` 就是用来解决这个顺序问题的。

---

### 8. Spring Cloud组件全景与选型

#### 组件间的调用关系（一次请求的全链路）

```
用户请求
  → Gateway (路由匹配 + JWT 校验 + 限流)
    → Service A (Nacos Discovery 获取地址 + LoadBalancer 选实例)
      → Feign → Service B
        → Sentinel (熔断保护)
          → Service B 实例
```

#### Nacos 注册中心的内部

**服务注册**：服务启动 → Application 启动完成事件 → `NacosAutoServiceRegistration.start()` → `NacosNamingService.registerInstance()` → 发 HTTP POST 给 Nacos Server

**健康检查**：Nacos 支持两种模式
1. 临时实例（默认）：客户端心跳（5s 一次），Nacos 15s 没心跳 → 不健康 → 30s 摘除
2. 持久实例：Nacos 主动健康检查（TCP/HTTP/MYSQL），用于无法引入 Nacos SDK 的老服务

**为什么 Nacos 配置中心是 CP、注册中心是 AP**：
- 配置管理需要强一致性（配置不一致可能导致重大故障），使用 Raft 协议
- 服务注册允许短暂不一致（网络抖动不应导致服务不可用），使用客户端心跳 + 服务端自动摘除

---

### 9. 服务网关（Gateway）的作用与实现：路由转发、权限控制、限流熔断？如何优化网关性能？

#### 是什么

API 网关是微服务架构的**统一入口**。它不是简单的反向代理，而是整个分布式系统的「前门」——所有外部请求必须经过网官才能到达内部服务。

网关承担了**横切关注点**：路由、鉴权、限流、熔断、日志、协议转换、灰度路由。这些功能如果每个微服务自己实现，会有大量重复代码和配置不一致问题。

**Gateway vs Zuul 1.x 的本质区别（面试常问）**：
- Zuul 1.x 基于 Servlet 2.5（阻塞 IO），每个请求一个线程 → 线程池打满 → 雪崩
- Gateway 基于 Spring WebFlux + Reactor（非阻塞 IO），基于 Netty 的 EventLoop 模型 → 少量线程处理大量连接。连接数可以轻松到 10000+ 而线程数只有 CPU 核数

#### 为什么要有网关

1. **安全防线**：外部请求不能直达内部服务，网关是第一道鉴权和攻击防护（SQL 注入、XSS 过滤）
2. **协议统一**：外部 HTTPS + JSON，内部可能是 gRPC + Protobuf，网关做协议翻译
3. **流量治理**：限流、熔断、灰度发布，集中管理比分散在每个服务更可靠
4. **前端友好**：后端微服务可能拆得细碎（一个页面需要调 5-6 个服务），网关做聚合（BFF 模式）

#### 底层原理

Gateway 三大核心组件：

**① Route（路由）**：`RouteDefinition` 包含 id、uri（目标地址）、predicates（匹配条件）、filters（过滤器链）。Route 定义可以来自 YAML 配置、Java DSL、或动态路由（从注册中心/Nacos 自动生成）。

**② Predicate（断言）**：`ServerWebExchange` → 断言检查 → 匹配则转发。本质是 Java 8 的 `Predicate<ServerWebExchange>`。内置：Path、Method、Header、Query、Cookie、Host、Before/After/Between（时间）。自定义 Predicate 只需实现 `RoutePredicateFactory`。

**③ Filter（过滤器）**：`GatewayFilter`（作用于单个路由）和 `GlobalFilter`（作用于所有路由）。过滤器链执行顺序由 Order 决定，数字越小优先级越高。`NettyRoutingFilter` 的 Order 是 `LOWEST_PRECEDENCE`（接近 Integer.MAX_VALUE），自定义 Filter 设较小 Order 就排在它前面。

**一次请求的完整处理链路**：
```
请求 → DispatcherHandler
  → RoutePredicateHandlerMapping（匹配路由）
    → FilteringWebHandler（组装过滤器链 + 执行）
      → 前置 GlobalFilter（Order 小 → 大）
        → 路由级 GatewayFilter Sequence
          → NettyRoutingFilter（实际转发请求到下游）
        → 后置 GatewayFilter
      → 后置 GlobalFilter
```

#### 各功能的实现细节

**路由转发**：ProxyExchange 把请求属性（Method、Headers、Body）从原始请求复制到下游请求。支持 `lb://service-name` 走负载均衡（Spring Cloud LoadBalancer）。

**权限控制（JWT + 全局过滤器）**：
```java
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        // 1. 解析 JWT → 校验签名和过期时间
        // 2. 从 claims 中提取 userId + roles
        // 3. 写入 Header 传递给下游：exchange.getRequest().mutate().header("X-User-Id", userId)
        // 4. 与 URL 的权限规则对比（从配置中心加载）
        // 5. 不合法 → 直接返回 401，不转发
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        return response.setComplete();
        // 合法 → chain.filter(exchange)
    }
}
```

关键设计：**网关只做 Token 校验（验证是否有效），不做权限校验（验证是否有权限访问具体资源）**。权限校验下沉到业务服务（网关不知道每个服务的权限模型）。网关做权限校验 = 网关耦合了所有服务的权限逻辑 = 每次加接口都要改网关。

**限流**：`RequestRateLimiter` GatewayFilter + `KeyResolver`。KeyResolver 决定用什么维度限流（IP、userId、接口路径）。核心是 Lua 脚本在 Redis 中原子操作令牌桶：
```lua
-- 令牌桶算法
local tokens_key = KEYS[1]
local timestamp_key = KEYS[2]
local rate = tonumber(ARGV[1])     -- 令牌生成速率
local capacity = tonumber(ARGV[2])  -- 桶容量
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local last_tokens = tonumber(redis.call('get', tokens_key))
if last_tokens == nil then last_tokens = capacity end

local last_refreshed = tonumber(redis.call('get', timestamp_key))
if last_refreshed == nil then last_refreshed = 0 end

local delta = math.max(0, now - last_refreshed)
local filled_tokens = math.min(capacity, last_tokens + (delta / 1000 * rate))
local allowed = filled_tokens >= requested
local new_tokens = filled_tokens
if allowed then new_tokens = filled_tokens - requested end

redis.call('setex', tokens_key, 60, new_tokens)
redis.call('setex', timestamp_key, 60, now)
return allowed and 1 or 0
```

**熔断**：集成 Resilience4j CircuitBreaker。在路由的 filter 中声明 `CircuitBreaker` 工厂，当下游服务错误率 > 阈值 → 熔断器打开 → 快速返回 fallback 响应（不转发请求到故障服务）。

#### 性能优化（架构师级）

1. **路由规则缓存在 Caffeine**（默认已开启）：路由匹配后缓存结果，避免每次重建 Route 对象
2. **GlobalFilter 精简**：只有真正全局的逻辑才放 GlobalFilter。每个额外 Filter 增加一次 `Mono.flatMap` 调用，虽然轻量但链路越长性能越差
3. **RequestBody 缓存优化**：如果多个 Filter 需要读 Body（如验签 + 日志收集），用 `AdaptCachedBodyGlobalFilter` 缓存 Body 避免重复读取
4. **WebClient 替代 RestTemplate**：在自定义 Filter 中需要调外部服务时，用 WebClient（异步非阻塞）而不是 RestTemplate（同步阻塞）。RestTemplate 会占用 Gateway 的少数 Netty 线程，导致整个网关吞吐下降
5. **网关水平扩展**：网关无状态，直接加 Pod。配合 K8s HPA（基于 CPU/内存自动扩缩）
6. **合理使用超时**：`connect-timeout` 和 `response-timeout`，防止慢服务拖垮网关的连接资源

#### 坑点

1. **不要用 Spring MVC + Gateway 混用**：Gateway 依赖 WebFlux（Reactor），引入 `spring-boot-starter-web` 会把 WebFlux 踢掉。症状：网关启动正常但路由不生效。
2. **GlobalFilter 中阻塞操作**：在 Filter 中调 `RestTemplate.getForObject()` → 同步阻塞 Netty EventLoop → 网关整体卡死。
3. **Order 冲突**：自定义 Filter 的 Order 和内置 Filter 冲突（如都在 Order=0）。内置的默认 Order 可以查 `org.springframework.cloud.gateway.filter.GatewayFilter` 的常量。

---

### 10. 配置中心（Nacos/Apollo）的实现原理？配置动态刷新、灰度发布、权限管理？配置同步失败如何解决？

#### 是什么

配置中心解决的是「配置散落在每个服务的 application.yml 中，改一个配置要重新打包部署全部服务」的问题。核心能力：**集中管理 + 动态推送 + 环境隔离 + 版本管理**。

#### 为什么需要配置中心（不只是"方便改配置"）

1. **动态变更**：限流阈值、开关、灰度比例 → 不能因为改个开关就重启 50 个微服务
2. **审计追溯**：谁在什么时候改了什么配置 → 出问题时能追溯到根因（是改配置导致的）
3. **灰度验证**：先在 1 个实例上验证配置是否正确，再全量推送
4. **多环境管理**：dev/staging/prod 的配置分开，避免开发环境的数据库地址被推到生产

#### Nacos 配置中心的底层实现

**配置存储**：Nacos Server 用 MySQL（生产）+ 内存缓存。配置数据的结构是 `namespace + group + dataId` 三元组。为什么需要三元组？namespace 隔离环境（dev/prod），group 隔离应用（用户服务/订单服务），dataId 是配置名称。

**客户端长轮询（Long Polling）机制**：
1. 客户端启动 → 拉取全量配置 + MD5 校验码 → 本地缓存（`LocalConfigInfoProcessor`）
2. 发起 HTTP 长轮询请求（Header 带 `Long-Pulling-Timeout: 30000`）→ 服务端挂起连接
3. 服务端在 30s 内检查配置是否变更（对比 MD5）
4. 有变更 → 立即返回新配置的 dataId + MD5；无变更 → 30s 超时返回空
5. 客户端收到变更 → 重新拉取该 dataId 的完整配置 → 触发本地刷新

**为什么用长轮询而不是 WebSocket 或短轮询**：
- 短轮询（每 1 秒查一次）→ 实时性差 + 对服务端压力大
- WebSocket → 需要网关/防火墙支持 WebSocket 协议（很多企业网络不允许长连接）
- 长轮询 → 本质是 HTTP（防火墙友好），实时性好（类似 push），服务端无变更时不返回数据（省带宽）

#### 配置动态刷新的实现

Spring Cloud 中动态刷新的两种方式：

**方式一：`@RefreshScope`**（针对需要刷新的单一 Bean）
```java
@RefreshScope  // 配置变更时重建此 Bean
@RestController
public class ConfigController {
    @Value("${switch.enabled}")
    private boolean enabled;
}
```

原理：`@RefreshScope` 的 Bean 被包装为代理对象（`ScopedProxyFactoryBean`），存在 `RefreshScope` 的缓存中。收到 `RefreshEvent` → `ContextRefresher.refresh()` → 清空 `RefreshScope` 的缓存 → 下次访问该 Bean 时重新创建（读取最新配置）。

**方式二：`@ConfigurationProperties`**（推荐）
```java
@ConfigurationProperties(prefix = "my.config")
@Component
public class MyConfigProperties {
    private boolean enabled;  // 自动绑定 + 刷新
    // getter/setter
}
```

`@ConfigurationProperties` 绑定的 Bean 不需要 `@RefreshScope` 也能刷新！因为 `ConfigurationPropertiesRebinder` 会在 `EnvironmentChangeEvent` 后重新绑定。但如果是 `@Bean` 方法中使用了配置值，对应的组件可能不会自动重建（需要手动 `@RefreshScope`）。

**详细刷新链路**：
```
Nacos 客户端感知配置变更
  → NacosContextRefresher 发布 RefreshEvent
  → RefreshEventListener 收到事件
  → ContextRefresher.refresh()
    → 清空 RefreshScope 缓存
    → 重新绑定 @ConfigurationProperties Bean
    → 发布 EnvironmentChangeEvent
```

**坑点**：数据库连接池（druid/HikariCP）、Redis 连接（Jedis/Lettuce）的配置变更不会自动重建连接！因为连接池在 `@Bean` 方法中创建，配置变更后需要重新调用 `@Bean` 方法，但 Spring 默认不会自动重建。解决方案：
```java
@Bean
@RefreshScope
public DataSource dataSource() { ... }  // 加了 @RefreshScope 会重建
```
但这样连接池会断开旧连接 + 建立新连接，对正在执行的 SQL 有不小的影响。更好的方式：连接池本身支持动态修改部分参数（如 Druid 的 `restart()` 方法）。

#### 灰度发布

**Nacos 的灰度**：创建配置时选择「Beta 发布」→ 输入目标 IP → 只有这个 IP 的客户端拉取灰度值 → 验证通过 →「正式发布」推送所有客户端。

**Apollo 的灰度更完善**：可以按 IP、AppId、标签维度灰度，支持「灰度发布」→「全量发布」→「回滚」。发布后有版本历史，支持秒级回滚到任意历史版本。

#### 配置同步失败排查

1. **检查 namespace + group + dataId 是否和配置中心上一致**（看似简单，实则 80% 的问题都是这个）
2. **检查客户端日志** → 搜 `add-listener` 确认长轮询是否正常。Nacos 客户端日志路径 `~/logs/nacos/config.log`
3. **检查网络** → Nacos Server 端口（默认 8848）+ gRPC 端口（默认 9848，+1000）。9848 容易被防火墙拦截
4. **检查 Nacos 版本兼容性** → 客户端和服务端版本不匹配（大版本升级后容易出现）
5. **检查内存** → 配置过多，Nacos 服务端内存不足，长轮询被拒绝

---

### 11. 微服务通信方式：RESTful、RPC、消息队列的选择？Feign 底层实现与优化？

#### 是什么

微服务间通信有三种模式：

- **同步请求/响应**：HTTP RESTful（Feign）、gRPC、Dubbo RPC → 适合查询（需要立即返回结果）
- **异步消息**：RocketMQ、Kafka、RabbitMQ → 适合事件通知（不需要立即返回结果）
- **响应式流**：RSocket、WebSocket → 适合实时推送（消息推送、股票行情）

#### 三种方式的完整对比

| 维度 | RESTful (Feign) | RPC (Dubbo) | 消息队列 (RocketMQ) |
|------|----------------|-------------|-------------------|
| 协议 | HTTP/1.1 + JSON | TCP + 私有协议 | TCP + 自定义协议 |
| 调用方式 | 同步/异步 HTTP | 同步/异步 RPC | 异步 |
| 序列化 | JSON（文本） | Hessian2/Protobuf（二进制） | 自定义 |
| 性能 | 一般（文本+HTTP头） | 高（二进制+私有协议） | 高（顺序写磁盘） |
| 耦合度 | 低（HTTP 接口） | 中（接口依赖） | 低（消息格式） |
| 跨语言 | 天然支持 | 需要多语言 SDK | 天然支持 |
| 适用 | 对外 API、跨部门 | 内部高性能调用 | 异步解耦、削峰 |

#### Feign 底层实现（源码级别）

Feign = **接口注解 → 动态代理 → HTTP 请求** 的完整映射。

**步骤 1：@FeignClient 扫描**
```java
// Spring Cloud 在启动时扫描 @FeignClient 注解
@FeignClient(name = "user-service", url = "http://user-service")
public interface UserClient {
    @GetMapping("/api/user/{id}")
    User getUser(@PathVariable("id") Long id);
}
```

**步骤 2：FeignClientFactoryBean → 创建 JDK 动态代理**

每个 `@FeignClient` 接口对应一个 `FeignClientFactoryBean`（FactoryBean 模式），`getObject()` 返回 JDK 动态代理对象。

**步骤 3：ReflectiveFeign 构建 MethodHandler**

把接口的每个方法绑定一个 `MethodHandler`（封装了 HTTP 请求的所有细节：URL 模板、Method、Headers、Body 编码器、返回值的 Decoder）。这是 Feign 的核心抽象——**一个 Java 方法 = 一个 HTTP 请求模板**。

**步骤 4：MethodHandler.invoke() → 构建 HTTP Request → 发送 → 解析 Response**

```java
// 调用 client.getUser(1L) 时实际执行：
MethodHandler.invoke(args) {
    // 1. 解析方法参数 → 结合 URL 模板生成完整 URL
    RequestTemplate template = new RequestTemplate();
    template.method("GET");
    template.uri("/api/user/{id}").resolveMap(Map.of("id", 1L));
    
    // 2. 通过 Client 发送 HTTP 请求
    Request request = target.apply(template);
    Response response = client.execute(request, options);
    
    // 3. 解码 Response
    return decoder.decode(response, User.class);
}
```

**步骤 5：LoadBalancer 集成**

当 URL 是 `http://user-service` 时，LoadBalancer 拦截 `Client.execute()`，从 Nacos 拿到 `user-service` 的实例列表 → 选一个 `192.168.1.x:8080` → 替换 URL → 发送。

#### Feign 性能优化（6 个维度）

**1. 连接池（最重要）**：默认 `HttpURLConnection` 不支持连接池，每次请求新建连接（三次握手 + TCP 慢启动）。
```yaml
feign:
  okhttp:
    enabled: true       # 替换为 OkHttp（支持连接池）
# 或
  httpclient:
    enabled: true       # 替换为 Apache HttpClient
    max-connections: 200
    max-connections-per-route: 50
```

**2. 超时配置（防止雪崩）**：
```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 1000     # 连接超时 1s
        readTimeout: 3000        # 读取超时 3s
```
注意：connectTimeout 是连接建立的超时（TCP 握手），readTimeout 是等待响应的超时。商品详情这种需要调多个服务的接口，readTimeout 设太短会导致超时假阳性。

**3. 数据压缩**：开启 Gzip → 大量文本数据时压缩率 80%+，省带宽，但增加 CPU 开销。
```yaml
feign:
  compression:
    request:
      enabled: true
      mime-types: text/xml,application/json
      min-request-size: 2048
```

**4. 批量合并**：N 个独立查询合并为 1 个批量接口。
```java
// 不好：循环调 Feign
for (Long id : ids) { userClient.getUser(id); }   // N 次 HTTP 调用

// 好：批量接口
List<User> users = userClient.getUsersByIds(ids);  // 1 次 HTTP 调用
```

**5. 缓存**：不常变的数据（省份列表、配置项）加本地缓存（Caffeine），避免每次调 Feign。

**6. 避免大对象传输**：返回大列表时，Feign 要把整个 JSON 反序列化到内存。如果返回 10 万条记录 → Feign 内存可能爆。用分页或流式传输。

#### 坑点

1. **Feign 默认超时 1s**：业务稍慢就超时 → 配置 `readTimeout`
2. **Feign 不支持文件流**：文件传输（上传/下载大文件）Feign 很痛苦（需要全量读入内存）。建议走直连 HTTP 或 OSS 直传
3. **Feign + Sentinel 的 fallback 不生效**：需要在 `@FeignClient` 中指定 `fallback` 或 `fallbackFactory`（工厂模式可以拿到异常信息做分类处理）
4. **RequestInterceptor 中做耗时操作**：同步查库拿 Token → 每次 Feign 调用都查库 → 拖慢所有 Feign 请求

---

### 12. 服务治理包含哪些方面？如何落地？

#### 是什么

服务治理不是某个具体的工具，而是**保障微服务系统稳定运行的一整套体系**。类比：城市管理（交通管制、垃圾回收、安全巡逻、医疗急救）就是一种治理体系。

#### 五大领域

**① 服务注册与发现（"白页"）**
- 功能：服务启动注册、下线摘除、健康检查、元数据管理
- 技术：Nacos Eureka Consul
- 关键指标：摘除延迟（服务挂了多久后调用方停止调它）= 心跳间隔 × 允许失败次数

**② 配置管理（"控制面板"）**
- 功能：集中管理、动态推送、版本管理、灰度发布、审计
- 技术：Nacos Apollo Spring Cloud Config
- 关键：必须支持回滚！改配置改出问题，能一秒回滚到上一个版本

**③ 流量管理（"红绿灯"）**
- 功能：负载均衡、限流、熔断、降级、灰度路由
- 技术：Sentinel Resilience4j Spring Cloud LoadBalancer
- 分层：网关层限流（防外部攻击）+ 服务层限流（防内部雪崩）+ 热点参数限流（防止单一热点拖垮）

**④ 可观测性（"监控探头"）**
- 链路追踪：SkyWalking（追踪请求完整路径，定位慢服务）
- 指标监控：Prometheus + Grafana（JVM/CPU/QPS/RT/错误率/业务指标）
- 日志中心：ELK（统一格式 + traceId 串联 + 告警规则）

**⑤ 稳定性保障（"保险机制"）**
- 超时：所有 RPC 调用必须设超时（不设超时 = 一条慢调用拖垮整个线程池）
- 重试：只重试幂等操作（查询可以重试，扣款不能重试）
- 幂等：所有写接口必须是幂等的（靠唯一键、乐观锁、状态机保证）
- 优雅上下线：K8s + Spring Boot 优雅关闭 + 注册中心摘除

#### 落地的分阶段策略（不是一步到位）

```
第一阶段（基础）：注册中心 + 配置中心
  目标：服务能发现彼此、配置能集中管理
  工具：Nacos（二合一，省运维成本）

第二阶段（保命）：限流 + 熔断 + 降级
  目标：一个人炸了不拖死全团
  工具：Sentinel（Dashboard 控制台 + 规则持久化）
  关键：先上核心链路（下单、支付），再逐步覆盖全系统

第三阶段（看清楚）：链路追踪 + 监控 + 日志
  目标：出问题能快速定位
  工具：SkyWalking（Java Agent 零侵入）+ ELK（Filebeat 自动采集）
  关键：traceId 在全链路透传（Gateway → Service A → Service B → DB/MQ）

第四阶段（自动化）：CI/CD + 自动扩缩容 + 故障自愈
  目标：少人工介入
  工具：K8s HPA + Prometheus + AlertManager 自动触发扩缩
```

#### 服务治理的标准化（架构师的核心工作）

不是买了 Nacos + Sentinel + SkyWalking 就叫服务治理。关键是对团队输出**标准**：

1. 统一日志格式：`[traceId] [userId] [ip] [method] [msg]`  → 方便 ELK 搜索
2. 统一异常处理：`@RestControllerAdvice` + 统一错误码 `Result<T>` → 网关知道每个错误是什么意思
3. 统一 traceId 传递：Gateway 生成 traceId → MDC → Feign Interceptor 透传 → MDC → 日志自动带 traceId
4. 统一超时配置：查询 < 3s，写入 < 5s，导出 < 30s
5. 统一限流阈值文档：每个接口的限流值有据可查，不是拍脑袋设的数字

---

### 13. 分布式追踪系统（SkyWalking）的作用？如何集成与定位跨服务问题？

#### 是什么

分布式追踪的核心是**把一个请求在 N 个服务间的调用串成一条链**。单服务时代，一个请求的日志全在一个文件里；微服务时代，一个请求可能经过 Gateway → A → B → C → D → DB → MQ，日志散落在 6 个服务中。没有追踪，你只能一个服务一个服务去翻日志找。

**Trace vs Span**：
- **Trace**：一次完整请求的调用链（从 Gateway 到最后一个服务），是多个 Span 的集合
- **Span**：一次具体操作（如一次 Feign 调用、一次 SQL 查询），包含开始时间、结束时间、操作名、标签（HTTP 状态码、SQL 语句）

#### 为什么需要（不只是"定位慢调用"）

1. **故障爆炸半径判定**：某个服务出问题，影响了多少上游调用方？SkyWalking 拓扑图一眼看出
2. **性能瓶颈精确定位**：不是"这个接口慢"，而是"这个接口在调用户服务的 `getUserAddress` 方法时等待了 2.3s"
3. **依赖梳理**：50+ 微服务谁调了谁？哪些是循环依赖？拓扑图就是最好的文档
4. **容量规划**：哪个服务 QPS 最高、哪个服务最热（调它的服务最多）→ 扩容优先级

#### SkyWalking 的底层原理（Java Agent 探针）

**核心机制：ByteBuddy 字节码增强 + Java Agent**

1. **Agent 启动**：`-javaagent:skywalking-agent.jar` → JVM 启动时调用 `premain()` 方法
2. **字节码增强**：Agent 在类加载时用 ByteBuddy 修改字节码：
   - 标注了 `@RequestMapping` 的方法 → 创建 EntrySpan
   - Feign Client 的方法 → 创建 ExitSpan（标记跨服务调用）
   - JDBC 的 `executeQuery()` → 创建 LocalSpan（标记本地操作）
3. **ContextCarrier**：调用下游时，把 TraceContext（traceId, segmentId, spanId）塞进 HTTP Header（如 Feign Interceptor）或 Dubbo Attachment
4. **Span 数据发送**：通过 gRPC 异步发送到 OAP Server（数据收集 + 聚合分析）

**为什么是 Agent 而不是 SDK**：
- SDK 侵入：每个服务加依赖 + 改代码 → 30 个服务改到疯
- Agent 无侵入：启动脚本加参数 → 不改代码 → 90% 的场景够用了

**自定义 Span 的场景**：Agent 自动埋的 Span 粒度不够时，用 `@Trace` 注解或 `TraceContext` API 手动创建 Span。

#### 集成方式

```bash
# 启动脚本加 agent 参数
java -javaagent:/opt/skywalking/skywalking-agent.jar \
     -DSW_AGENT_NAME=order-service \
     -DSW_AGENT_COLLECTOR_BACKEND_SERVICES=skywalking-oap:11800 \
     -jar order-service.jar
```

K8s 中使用 InitContainer 注入 agent：
```yaml
initContainers:
  - name: skywalking-agent
    image: apache/skywalking-java-agent:9.0.0
    command: ['cp', '-r', '/skywalking/agent', '/agent']
    volumeMounts:
      - name: agent
        mountPath: /agent
```

#### 实战排查案例

**问题**：订单详情接口 P99 > 5s

**排查**：SkyWalking → 找到这条慢 Trace → 时间线显示：
- Gateway → order-service: 50ms
- order-service → user-service: 60ms
- order-service → product-service: 80ms
- **order-service → price-service: 4.8s** ← 瓶颈！
- price-service: SQL `SELECT * FROM price_history WHERE ...` 全表扫描 400 万行

**定位到具体 SQL**：SkyWalking 的 LocalSpan 中记录了 JDBC 的 SQL 和执行耗时，直接看到这个 SQL 占了 4.4s。

**修复**：加联合索引 → price-service 耗时降到 10ms → 总接口耗时降到 200ms。

#### 坑点

1. **Agent 版本和 OAP 版本不匹配** → 追踪数据不显示
2. **大量自定义 Span** → 增加 CPU 开销（每个 Span 都要记录 + 序列化 + gRPC 发送），只需关键方法加 Span
3. **采样率**：SkyWalking 默认 100% 采样（所有请求都追踪），高 QPS 下 OAP Server 压力大。生产建议对正常的快请求降采样，慢请求全量保留

---

### 14. Spring Security的核心原理，如何实现认证和授权？项目中如何自定义JWT认证？

#### 是什么

Spring Security 是 Spring 生态的安全框架，核心是**过滤器链 + 认证（Authentication）+ 授权（Authorization）**。它不是单个组件，而是一系列 Security Filter 按顺序组成的过滤器链。

#### 为什么需要

1. **安全是一个横切关注点**：不能每个 Controller 手动校验 Token
2. **标准化认证流程**：支持表单登录、OAuth2.0、JWT、SAML、LDAP 等多种认证方式
3. **防御常见攻击**：CSRF、Session Fixation、Clickjacking、XSS 等内置防护

#### 底层原理（过滤器链 + 认证 + 授权）

**过滤器链架构**：
```
请求 → SecurityFilterChain
  → SecurityContextPersistenceFilter (从 Session 恢复 SecurityContext)
  → CsrfFilter (CSRF 校验)
  → ... (各种自定义 Filter)
  → UsernamePasswordAuthenticationFilter (处理登录请求)
  → ... 
  → FilterSecurityInterceptor (授权：检查是否有权限访问此 URL)
  → ExceptionTranslationFilter (处理认证/授权异常)
```

**认证（Authentication）流程**：
1. `AuthenticationFilter` 从请求中提取凭证（用户名+密码 / JWT Token）
2. 构造 `Authentication` 对象（`UsernamePasswordAuthenticationToken`，状态=未认证）
3. 交给 `AuthenticationManager`（实际是 `ProviderManager` 委托给多个 `AuthenticationProvider`）
4. `AuthenticationProvider` 中的 `authenticate()` 方法：
   - `DaoAuthenticationProvider`：调 `UserDetailsService.loadUserByUsername()` 从数据库加载用户 → `PasswordEncoder.matches()` 验证密码
   - `JwtAuthenticationProvider`（自定义）：解析 JWT → 验签 → 提取用户信息
5. 认证成功 → 返回已认证的 `Authentication` → 存入 `SecurityContextHolder`（默认 ThreadLocal）
6. 认证失败 → 抛 `AuthenticationException` → `AuthenticationFailureHandler` 处理

**授权（Authorization）流程**：
1. `FilterSecurityInterceptor` 拦截请求
2. 从 `SecurityContextHolder` 获取当前用户的 `Authentication`（含角色/权限）
3. `AccessDecisionManager` 投票：
   - `AffirmativeBased`（默认）：一票同意即通过
   - `ConsensusBased`：多数票原则
   - `UnanimousBased`：全部同意才通过
4. `AccessDecisionVoter`：`RoleVoter`（`hasRole('ADMIN')`）、`AuthenticatedVoter`（`isAuthenticated()`）等
5. 通过 → 放行；不通过 → 抛 `AccessDeniedException`

#### 项目中的 JWT 自定义认证实现（完整方案）

**步骤 1：自定义 JWT 过滤器**
```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                     HttpServletResponse response, 
                                     FilterChain filterChain) {
        String token = extractToken(request);  // 从 Authorization Header 提取
        if (token != null && jwtUtil.validateToken(token)) {
            // 解析 JWT 获取 userId、roles
            String userId = jwtUtil.getUserId(token);
            List<GrantedAuthority> authorities = jwtUtil.getRoles(token).stream()
                .map(SimpleGrantedAuthority::new).collect(Collectors.toList());
            
            // 构造已认证的 Authentication 放入 SecurityContext
            UsernamePasswordAuthenticationToken authentication =
                new UsernamePasswordAuthenticationToken(userId, null, authorities);
            authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }
        filterChain.doFilter(request, response);
    }
}
```

**步骤 2：配置 SecurityFilterChain**
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) {
    http
        .csrf().disable()                                      // 无状态 API 不需要 CSRF
        .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)  // 无状态
        .and()
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/login", "/api/register").permitAll()  // 登录注册白名单
            .requestMatchers("/api/admin/**").hasRole("ADMIN")           // 管理员接口
            .anyRequest().authenticated()                                 // 其余需要认证
        )
        .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);
    return http.build();
}
```

**步骤 3：登录接口生成 JWT**
```java
@PostMapping("/api/login")
public Result<String> login(@RequestBody LoginRequest req) {
    // 1. 验证用户名密码
    User user = userService.findByUsername(req.getUsername());
    if (!passwordEncoder.matches(req.getPassword(), user.getPassword())) {
        throw new BadCredentialsException("密码错误");
    }
    // 2. 生成 JWT（包含 userId、roles、过期时间）
    String token = Jwts.builder()
        .setSubject(user.getId().toString())
        .claim("roles", user.getRoles())
        .setExpiration(new Date(System.currentTimeMillis() + 7200000))
        .signWith(SignatureAlgorithm.HS256, secretKey)
        .compact();
    return Result.success(token);
}
```

#### 坑点

1. **Token 刷新问题**：JWT 一旦签发，在过期前无法撤销（无状态的设计取舍）。解决：短有效期的 Access Token（15 分钟）+ 长有效期的 Refresh Token（7 天）
2. **SecurityContextHolder 跨线程**：默认 MODE_THREADLOCAL，新建线程时丢失认证信息。解决：`SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL)` 或使用 `DelegatingSecurityContextExecutor`
3. **Filter 顺序**：认证 Filter 必须放在授权 Filter 前面。`@Order` 设置不对 → 认证未执行就被授权拦截 → 全部 403
4. **`permitAll()` 的请求仍然会走 Filter 链**：只是 `FilterSecurityInterceptor` 放行，前面的认证 Filter 还是会解析 Token。白名单请求应该跳 Token 解析

---

### 15. 微服务架构下的跨域问题如何解决？

#### 是什么

跨域（CORS, Cross-Origin Resource Sharing）是浏览器的同源策略限制：协议、域名、端口三者任一不同就是跨域。浏览器拦截跨域请求的响应，除非服务器明确返回 `Access-Control-Allow-Origin` 头。

这不是后端的问题，是**浏览器**的安全策略。Postman 测跨域接口不会报跨域错误（Postman 不是浏览器）。

**简单请求 vs 预检请求**：
- 简单请求（GET/POST + 标准 Header）：浏览器直接发，服务器返回 `Access-Control-Allow-Origin` 则成功
- 复杂请求（PUT/DELETE + 自定义 Header）：浏览器先发 OPTIONS（预检请求），服务器返回 `Access-Control-Allow-Methods` 等允许信息后，再发正式请求

#### 为什么微服务中有跨域问题

1. **前后端分离**：前端 `localhost:3000` → 后端 `api.example.com`
2. **多域名**：PC `www.example.com` + H5 `m.example.com` → 同一 API
3. **微服务多端口**：本地开发时 service-a:8081, service-b:8082

#### 解决方案的三种层次

**方案一：网关层统一处理（推荐）**

原理：所有请求进网关，网关统一加 CORS 响应头。一次配置解决所有服务。
```yaml
spring:
  cloud:
    gateway:
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins: "https://www.example.com,https://m.example.com"
            allowedMethods: GET,POST,PUT,DELETE,OPTIONS
            allowedHeaders: "*"
            allowCredentials: true
            maxAge: 3600  # 预检请求缓存时间
```
注意：`allowCredentials=true` 时，`allowedOrigins` 不能是 `*`（必须是具体域名）。这是浏览器强制规范。

**方案二：Nginx 反向代理（终极方案）**

前端和 API 用同一域名，Nginx 根据路径代理到不同后端：
```nginx
server {
    listen 80;
    server_name www.example.com;
    location /api/ {
        proxy_pass http://gateway:8080;
    }
    location / {
        proxy_pass http://frontend:3000;
    }
}
```
前端请求 `/api/order` → 同域名 → 不跨域！Nginx 本质上把跨域问题消除了。

**方案三：单个服务解决**（临时方案，Gateway 前的过渡）
```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
            .allowedOrigins("https://www.example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .allowCredentials(true);
    }
}
```

#### 坑点

1. **OPTIONS 预检请求的缓存**：`Access-Control-Max-Age` 设太短 → 每次调 API 都发两次请求（OPTIONS + 正式）→ 增加延迟。建议设 3600s
2. **自定义 Header 触发预检**：`Authorization` Token、`X-Trace-Id`、`X-User-Id` 都是非简单 Header → 必定触发预检。减少不必要的自定义 Header
3. **`@CrossOrigin` 注解不生效**：被 Spring Security 的 Filter 链提前拦截（Security Filter 在 MVC 之前执行）。应配在 Security 的 `cors()` 中

---

### 16. 微服务的平滑升级（无停机升级）如何实现？蓝绿部署、金丝雀发布的流程？

#### 是什么

平滑升级的本质是：**旧版本实例仍在处理请求，新版本实例逐步替换旧版本，外部用户无感知切换**。理想状态下：零停机、零感知、秒级回滚。

#### 为什么传统部署做不到平滑升级

传统部署：停服务 → 换 jar/war → 启服务。停机时间 = 旧的关闭时间 + 新的启动时间 = 30s - 几分钟。这 30 秒里，所有请求全部失败。

#### 三种升级策略的深度对比

**① 滚动发布（Rolling Update，K8s 默认）**

原理：逐个替换 Pod。新 Pod 就绪（readinessProbe 通过）→ 击杀一个旧 Pod → 循环直到全部替换。

```yaml
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # 最多超出 replicas 1 个 Pod（升级时 5 个）
      maxUnavailable: 0   # 至少 4 个可用（0 不可用 → 0 停机）
```

优点：K8s 原生支持，配置简单。缺点：回滚慢（`kubectl rollout undo` 逐个回滚 Pod）。适合：普通业务服务。

**② 蓝绿部署（Blue-Green）**

原理：两套完整环境（蓝=当前版本，绿=新版本）。绿部署好 → 验证通过 → 网关/LB 一把切换流量到绿。

```
蓝（旧版本 4 Pod）← 当前流量
绿（新版本 4 Pod）← 部署完成，等待切换

Step 1: 切换入口流量（Nginx upstream → 改为绿 IP）
Step 2: 观察绿是否正常
Step 3: 正常 → 保留绿、回收蓝
      异常 → 切回蓝（秒级回滚）
```

优点：秒级回滚（切回蓝即可）、新旧版本完全隔离。缺点：资源翻倍（需要两套环境同时存在）。适合：支付、金融等不允许任何风险的核心业务。

**③ 金丝雀发布（Canary）**

原理：先部署 1-2 个新版本实例 → 引入少量流量（5%）→ 观察无异常 → 逐步增加新版本比例（25% → 50% → 100%）。

实现方式（Nacos + Gateway）：
```
1. 新版 Pod 启动时注册到 Nacos，metadata 加 version=v2
2. Gateway Filter 根据请求头或用户 ID hash 决定路由到 v1 还是 v2
   - userId % 100 < 5 → 打 v2（5% 流量）
   - 其余 → 打 v1（95% 流量）
3. 观察 SkyWalking 上 v2 的错误率和 RT
4. 正常 → 调大比例 → 最终 100% v2
```

优点：风险最低（出问题只影响 5% 用户）、用真实流量验证。缺点：发布慢（逐步扩大需数小时）、需要流量分配能力。适合：C 端高流量应用。

#### 优雅上下线的关键技术点

**优雅下线（PreStop Hook + Spring Boot）**：
```yaml
lifecycle:
  preStop:
    exec:
      command:
        - /bin/sh
        - -c
        - |
          # 1. 从注册中心摘除（不再接收新请求）
          curl -X DELETE http://nacos:8848/nacos/v1/ns/instance?... 
          # 2. 等待已有请求处理完
          sleep 30
          # 3. Spring Boot 优雅关闭
```

**Spring Boot 优雅关闭配置**：
```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

**优雅上线（ReadinessProbe）**：
```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 20   # 启动后等 20 秒再检查
  periodSeconds: 5
  failureThreshold: 3       # 失败 3 次才标记 NotReady
```

只在连接池初始化、缓存预热、注册中心注册全部成功后，`/actuator/health/readiness` 才返回 UP。K8s 确认 READY 后才把流量引入这个 Pod。

#### 坑点

1. **K8s 先杀旧 Pod 再更新 Service 的 iptables**：即使 Pod 被标记 Terminating，到 Service 的 iptables 生效还有几秒延迟 → 请求被路由到正在关闭的 Pod → 502 错误。解决：preStop 先摘除 + sleep 30s
2. **数据库 Schema 变更的兼容性**：平滑升级要求新旧版本同时存在。DML 必须前向兼容（v1 代码能跑在 v2 的 Schema 上）。方法是：先加列（允许 NULL）→ 部署 v2 → 确认 v1 下线 → 删旧列
3. **回滚不是免费的**：滚动回滚需要时间（每个 Pod 几十秒），如果新版上线后大量 500，这段时间里业务是挂的。所以必须配熔断降级

#### 平滑升级

**蓝绿 vs 金丝雀 vs 滚动**：
- 蓝绿：资源浪费 2 倍，但回滚秒级（最适合金融 / 支付）
- 金丝雀：资源低浪费，发布慢但安全（C 端应用首选）
- 滚动：实现最简单，K8s 原生支持，但回滚慢（逐步替换旧 Pod）

---

