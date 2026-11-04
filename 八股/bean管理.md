# Bean 管理

## 📑 索引 / 目录

**大节**

- [[#一、Bean 的定义与注册]]
- [[#二、Bean 的依赖注入]]
- [[#三、Bean 的作用域]]
- [[#四、Bean 的生命周期]]
- [[#五、Bean 的高级特性与底层原理]]

**正文**

- [[#什么是 Bean？它能干什么？]]
- [[#bean 的依赖注入]]
- [[#bean 的注解方式]]
- [[#补充方式：@Configuration + @Bean]]
- [[#条件装配：@Conditional、@Profile]]
- [[#Bean 的注入方式（构造器、Setter、字段）]]
- [[#解决多个同类型 Bean 的冲突]]
- [[#Bean 的作用域（正文）]]

> [!tip] 用法
> 上方索引全部可点击跳转。正文小节「底层原理」出现多次，统一命名时会带上所在章节，不会冲突。

---

#### 一、Bean 的定义与注册

1. XML 配置方式（老式写法）
2. 注解方式（主流写法）：`@Component`、`@Service`、`@Controller`、`@Repository`
3. Java 配置类方式：`@Configuration` 配合 `@Bean`
4. 组件扫描：`@ComponentScan` 的原理
5. 条件装配：`@Conditional`、`@Profile`

#### 二、Bean 的依赖注入

1. 注入方式：构造器注入、Setter 注入、字段注入（`@Autowired`）
2. 解决多个同类型 Bean 的冲突：`@Primary`、`@Qualifier`、`@Resource`
3. 依赖注入的底层原理

#### 三、Bean 的作用域

1. singleton（单例，默认）
2. prototype（多例）
3. Web 作用域：`request`、`session`
4. 单例 Bean 的线程安全问题

#### 四、Bean 的生命周期

1. 生命周期四大阶段：实例化 -> 属性赋值 -> 初始化 -> 销毁
2. 生命周期回调钩子：`@PostConstruct`、`InitializingBean` 等
3. 后置处理器（底层核心）：`BeanPostProcessor`
4. 生命周期底层源码流程

#### 五、Bean 的高级特性与底层原理

1. 循环依赖问题（A 依赖 B，B 依赖 A，三级缓存怎么解决？）
2. Bean 的代理（AOP 是怎么生成代理对象的？）
3. `ApplicationContext.getBean()` 的用法

---

## 什么是 Bean？它能干什么？

### 1. 什么是 Bean？（通俗比喻）

在传统编程里，我们需要一个对象时，都是自己动手：`EmployeeService employeeService = new EmployeeServiceImpl();`。这就好比你要造一辆车，得自己去炼钢、打螺丝、组装零件，非常累。

而在 Spring 的世界里，引入了 **Bean（豆子）** 这个概念。你可以把 Spring 想象成一个“超级智能大管家”**。你只需要告诉管家：“我需要一个员工服务对象”。管家就会在它的“大仓库”（IoC 容器）里帮你找，如果找到了，直接递给你；如果没有，它就会帮你创建好，存进仓库，然后再递给你。

**专业定义**：Bean 就是**由 Spring IoC 容器实例化、组装和管理的 Java 对象**。在这个大仓库里，所有的对象都被称为 Bean。

### 2. Bean 能干什么？（核心价值）

Bean 的存在是为了解决**解耦**和**简化开发**：

1. **控制反转（IoC）**：把创建对象的控制权交给了 Spring，你不再需要到处写 `new`。
2. **依赖注入（DI）**：Spring 会自动把 A Bean 塞进 B Bean 需要的地方，不用你手动组装。
3. **单例管理**：默认情况下，一个 Bean 在容器里只有一份（节省内存）。
4. **生命周期管理**：Spring 负责 Bean 的出生（实例化）、成长（初始化）、以及死亡（销毁）。
5. **功能增强（AOP）**：在不修改 Bean 代码的前提下，偷偷给它加上日志、事务等功能（比如你之前学到的事务管理）。

比如，你之前学的 `@RestController` 标注的 `EmployeeController`，启动项目后，它就不再是一个普通的 Java 类了，而是变成了由 Spring 管家管理的一个 Bean。

## bean 的依赖注入

**专业定义**：依赖注入（Dependency Injection，简称 DI）是 Spring 实现控制反转（IoC）的**核心手段**。它指的是：Spring 容器在创建 Bean 时，会自动将该 Bean 所依赖的其他 Bean（对象）传递（注入）给它。

比如在EmployeeServiceImpl要用到EmployeeMapper：

```java
@Service
public class EmployeeServiceImpl implements EmployeeService {
    @Autowired // 核心注解：告诉 Spring 帮我注入一个 EmployeeMapper
    private EmployeeMapper employeeMapper;
}
```

### 底层原理（依赖注入篇：Spring 在背后干了什么？）

**容器启动 -> 反射创建对象 -> 发现 `@Autowired` -> 去单例池按类型匹配找到依赖对象 -> 反射赋值。**

## bean 的注解方式

springboot2.5以前，只能使用XML方式，2.5以后注解方式成为主流

XML方式：就像你去人才市场招人，手里必须拿一份长长的名单（XML文件），上面写着“我要招张三当服务员，李四当收银员”。
注解方式：不再需要名单。你只需要在张三、李四的脑门上贴上标签（`@Service`、`@Repository`），然后 Spring 这个厂长拿个“扫描仪”（`@ComponentScan`）在项目里扫一圈，看到贴了标签的，就直接拉进工厂（IoC 容器）里管起来。

**四大基础注解** 用来标记不同类型的 Bean（本质上它们都包含了 `@Component`）：

1. **`@Component`**：通用注解，用在不好归类的地方。
2. **`@Controller` / `@RestController`**：用在 Web 控制层（如你之前遇到的 `EmployeeController`）。
3. **`@Service`**：用在业务逻辑层（如 `EmployeeServiceImpl`）。
4. **`@Repository`**：用在数据访问层（如 Mapper 接口或 DAO 类）。

代码示例：

```java
// 1. 控制层
@RestController // 告诉 Spring：这是处理 Web 请求的 Bean
public class EmployeeController {
    // ...
}

// 2. 业务层
@Service // 告诉 Spring：这是处理业务逻辑的 Bean
public class EmployeeServiceImpl implements EmployeeService {
    // ...
}

// 3. 数据层（如果用 MyBatis 的 @Mapper 或 @Repository）
@Repository
public interface EmployeeMapper {
    // ...
}
```

一般情况下，如果没有指定名字的话，Spring会把类名首字母小写，作为Bean的名字，如`EmployeeServiceImpl`变成`employeeServiceImpl`。当然也可以手动指定名字：

```java
@Service("myEmpService") // 这个名字就是以后 getBean() 时用的名字
public class EmployeeServiceImpl implements EmployeeService { ... }
```

## 补充方式：@Configuration + @Bean

如果我们要把一个**第三方库的类**（比如 RedisTemplate、ThreadPoolTaskExecutor）交给 Spring 管理，我们无法在那个类的源码上加 `@Component`。此时就需要写一个配置类：

```java
@Configuration // 声明这是一个配置类（它本身也是一个 Bean）
public class MyConfig {
    @Bean // 告诉 Spring：这个方法返回的对象，请存入容器
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        return executor;
    }
}
```

### 底层原理（@Configuration 篇：Spring 在背后干了什么？）

**`@ComponentScan` 划定范围 -> ASM 读字节码 -> 发现注解生成 BeanDefinition（图纸） -> 放入 Map -> 反射创建实例。**

**第三方类库**就好比去进了一批轮胎，放进自己的仓库里（Spring IoC容器），以后谁要用，直接去拿就行。
代码演示：

**没有 `@Bean` 时（传统方式）：**
每次你需要用 Redis，都要自己 `new`，还要配置参数：

```java
// 每次用到都要自己配一遍，极其繁琐，且浪费资源
RedisTemplate<String, Object> template = new RedisTemplate<>();
template.setConnectionFactory(factory);
template.setKeySerializer(new StringRedisSerializer());
// ... 然后才能使用
```

**有了 `@Bean` 时（Spring 方式）：**
你在配置类里写好放在config（配置类）包下，不指定名称，默认bean的名称就是方法名：

```java
@Configuration
public class RedisConfig {
    @Bean // 相当于把造好的 RedisTemplate 放进 Spring 车库
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);
        // 配置序列化器等
        return template;
    }
}
```

以后在业务层，你根本不需要去 `import` 第三方库的细节，也不需要去 `new`，只需要：

```java
@Autowired // Spring 自动从车库把你配好的 RedisTemplate 塞过来
private RedisTemplate<String, Object> redisTemplate;
```

**总结：`import` 是为了代码能编译通过（语法层面）；`@Bean` 是为了项目的对象能统一管理、复用和依赖注入（Spring 容器层面）。**

为啥要放在config包下？

这涉及到spring boot的包扫描机制（ComponentScan）：

默认扫描规则：
在启动类（默认在com包下的）有一个 `@SpringBootApplication` 注解，这个注解内部隐藏了 `@ComponentScan`。他的默认规则是：扫描主启动类所在包以及所有子包。如果定义的配置类不在本包下，可以在启动类加一个`@ComponentScan（"要扫描的包","原有的包"）`来指定扫描范围，加了注解后，默认`@SpringBootApplication`不再生效，所以在`@ComponentScan`里还要加原来的包。

生效原因：
因为RedisConfig在com.xx.config.RedisConfig在com.xx的子包下，主启动类一启动，就会扫描到这个配置类，然后解析里面的Bean，再把RedisTemplate造出来放进spring容器。

## 条件装配：@Conditional、@Profile

@Conditional（条件装配）：只有达到某个设定的满足条件，如在游戏中，收集到某些材料（满足条件），才能打造出某件隐藏装备（bean），条件不满足，这个bean就不存在。

@Profile（环境隔离）：比如只有佩戴“开发部”的工牌，才能进入开发部；只有佩戴“生产部”的工牌，才能进入生产部。代码只有一套，但根据当前激活的“环境”，决定加载哪套配置。

不使用条件装配的话，我们只要加了@Component或@Bean，Spring启动时就一定会把他们创建出来的，这和企业开发不符合，比如我写了一个 Redis 配置类，但如果某个同事的项目根本没引入 Redis 依赖，我希望这个配置类**自动失效**，不要报错。那要怎么办呢？
这就要用到条件装配了---只有满足了特定条件，spring才会把这个Bean放进容器里。
那要怎么使用呢？

### A：使用@Profile（最常用）

假设有两套数据库配置，一套连本地（dev），一套连云服务器（prod）

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("dev") // 只有当激活的环境是 dev 时，这个 Bean 才会被创建
    public DataSource devDataSource() {
        System.out.println("加载开发环境数据库...");
        return new DevDataSource();
    }

    @Bean
    @Profile("prod") // 只有当激活的环境是 prod 时，这个 Bean 才会被创建
    public DataSource prodDataSource() {
        System.out.println("加载生产环境数据库...");
        return new ProdDataSource();
    }
}
```

如何激活环境：
在application.yml里配置：

```yaml
spring:
  profiles:
    active: dev # 此时，Spring 只会加载 devDataSource 这个 Bean
```

### B：使用@Conditional（底层原理，springboot的基石）

@Profile其实就是@Conditional的特例，@Conditional更加灵活，可以自定义添加判断规则，
比如，写一个自定义条件：只有在项目引入`RedisTemplate` 这个类时，我才创建 `RedisConfig`：

```java
// 1. 定义一个条件类，实现 Condition 接口
public class RedisCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        try {
            // 尝试加载 RedisTemplate 类，加载成功返回 true，失败返回 false
            Class.forName("org.springframework.data.redis.core.RedisTemplate");
            return true;
        } catch (ClassNotFoundException e) {
            return false;
        }
    }
}

// 2. 在配置类上使用这个条件
@Configuration
@Conditional(RedisCondition.class) // 只有上面的 matches 返回 true，这个配置类才会生效
public class RedisConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate() { ... }
}
```

_注：Spring Boot 提供了大量现成的条件注解，比如 `@ConditionalOnClass`、`@ConditionalOnMissingBean`。Spring Boot 的“自动配置”魔法，底层全是靠 `@Conditional` 实现的。_

### 底层原理（条件装配篇：Spring 在背后干了什么？）

**条件装配就是 Spring 容器的“安检机”，在生成 BeanDefinition 之前，先检查条件，合格的放行，不合格的直接拦截。**

## Bean 的注入方式（构造器、Setter、字段）

### 代码实操与优缺点对比

**方式一：字段注入（最不推荐，但初学者最爱）**

```java
@Service
public class EmployeeServiceImpl implements EmployeeService {
    @Autowired // 直接贴在属性上
    private EmployeeMapper employeeMapper;
}
```

- **优点**：写起来极度简单，代码量最少，这也是为什么很多人一开始都用它。
- **缺点（为什么 IDEA 会画波浪线警告）**：
    - **无法加 `final`**：依赖不是不可变的，别人可以通过反射把它改成 `null`。
    - **隐藏依赖关系**：你只看类的构造函数，完全不知道这个类依赖了哪些东西。
    - **单元测试极其痛苦**：如果你想单独测试这个类（不启动 Spring），你必须用反射工具（如 `ReflectionTestUtils`）去强行把 `employeeMapper` 塞进去。
    - **容易引发空指针**：如果在构造器中使用这个依赖，会报 NPE（因为此时依赖还没注入）。

**方式二：构造器注入（Spring 官方最强推荐！）**

```java
@Service
public class EmployeeServiceImpl implements EmployeeService {
    private final EmployeeMapper employeeMapper;

    // Spring 4.3 之后，如果类只有一个构造器，@Autowired 可以省略不写！
    public EmployeeServiceImpl(EmployeeMapper employeeMapper) {
        this.employeeMapper = employeeMapper;
    }
}
```

- **优点**：
    - **依赖不可变**：加了 `final`，保证依赖在对象创建后不可更改，线程安全。
    - **完全初始化**：对象创建完成时，依赖必定已经准备好了，彻底杜绝空指针。
    - **暴露代码坏味道**：如果你的构造函数有十几个参数，你一眼就能看出这个类的职责太重了，需要重构。
    - **天然避免循环依赖**：A 依赖 B，B 依赖 A，Spring 启动时就会直接报错，而不是等到运行时才出问题。
- **缺点**：代码稍多。但如果你用 Lombok 的 `@RequiredArgsConstructor`（为所有 final 字段生成构造器），代码和字段注入一样简洁！

**方式三：Setter 注入（仅用于“可选依赖”）**

```java
@Service
public class EmployeeServiceImpl implements EmployeeService {
    private EmployeeMapper employeeMapper;

    @Autowired
    public void setEmployeeMapper(EmployeeMapper employeeMapper) {
        this.employeeMapper = employeeMapper;
    }
}
```

- **优点**：灵活，可以在对象创建后随时重新注入或更换依赖（比如把 Redis 换成 Memcached）。
- **缺点**：无法保证依赖在对象创建时一定存在（可能被调用时还没注入），依赖状态可变，容易引发线程安全问题。
- **适用场景**：**只有当某个依赖是“可选”的时候才用它。** 比如短信发送服务，有配置就发，没配置就不发。

### 实战最佳实践（必背）

- **强制依赖**（必须有的）：一律使用**构造器注入**。配合 Lombok 的 `@RequiredArgsConstructor` 使用，代码如下：

    ```java
    @Service
    @RequiredArgsConstructor // 自动生成包含所有 final 字段的构造器
    public class EmployeeServiceImpl implements EmployeeService {
        private final EmployeeMapper employeeMapper;
        // 直接使用 employeeMapper，Spring 会通过构造器注入
    }
    ```

- **可选依赖**（有就有，没有也能跑）：使用 **Setter 注入**。
- **绝对避免**在项目核心业务代码中使用**字段注入**。

## 解决多个同类型 Bean 的冲突

在代码里，假设 `EmployeeService` 接口有两个实现类：

```java
@Service
public class EmployeeServiceImplA implements EmployeeService { ... }

@Service
public class EmployeeServiceImplB implements EmployeeService { ... }
```

当我们写：

```java
@Autowired 
private EmployeeService employeeService;
```

时，Spring查到两个，直接报错，解决方法：

### 解决方案一：`@Primary`（设立默认首选）

**怎么用？** 在其中一个 Bean 上加 `@Primary`。Spring就会默认选它

```java
@Service
@Primary // 告诉 Spring：当有多个同类型 Bean 时，默认优先选我
public class EmployeeServiceImplA implements EmployeeService { ... }

@Service
public class EmployeeServiceImplB implements EmployeeService { ... }
```

- **优点**：全局生效，一劳永逸，适合确实有主次之分的场景。
- **缺点**：不够灵活。如果 B 类的使用频率其实更高，或者在某些特定场景非要用 B，`@Primary` 就不好使了。

### 解决方案二：`@Qualifier`（精确点名）

**怎么用？** 配合 `@Autowired` 使用，明确指定你要那个 Bean 的**名字**。

```java
@Autowired
@Qualifier("employeeServiceImplB") // 明确点名：我要名字叫 employeeServiceImplB 的那个
private EmployeeService employeeService;
```

- **优点**：非常精准，想用哪个点哪个，就近原则，非常灵活。
- **缺点**：需要知道 Bean 的确切名字（默认是类名首字母小写，也可以用 `@Service("myName")` 自定义）。
- **适用场景**：当不同业务方法需要不同实现类时

### 解决方案三：`@Resource`（Java 标准注解）

注意！`@Autowired` 是 **Spring 框架** 提供的，而 `@Resource` 是 **Java 标准（JSR-250）** 提供的。Spring 也支持它。

```java
@Resource(name = "employeeServiceImplA") // 直接按名字匹配
private EmployeeService employeeService;
```

如果不写 `name` 属性，`@Resource` 会默认尝试按照**属性名**去匹配 Bean，如果找不到，再退化为按类型匹配。

### 底层原理（冲突解决篇：Spring 在背后干了什么？）

`@Primary` 定海神针，`@Qualifier` 精确制导，`@Resource` 官方备用。优先用 `@Qualifier` 或 `@Resource` 明确指定，避免团队开发时产生困惑。

**🔥 面试高频：`@Autowired` vs `@Resource`**

- **来源不同**：`@Autowired` 是 Spring 的；`@Resource` 是 Java 官方标准（jdk 自带）。
- **匹配顺序不同**：`@Autowired` 默认**按类型（ByType）**匹配，发现多个才去找名字；`@Resource` 默认**按名字（ByName）**匹配，找不到名字才按类型。
- **注解位置**：`@Autowired` 可以写在构造器、Setter、字段上；`@Resource` 通常只写在字段和 Setter 上。

## Bean 的作用域（正文）

Spring 默认提供了多种作用域，但对于初学者来说，**最核心、最常用、面试最常考的只有两个：`singleton`（单例）和 `prototype`（多例）。**

### 通俗比喻（公司饮水机 vs 一次性纸杯）

- **Singleton（单例，默认）**：就像公司的**公共饮水机**。全公司所有人（所有代码）用的都是这一台。容器一启动，饮水机就造好了。谁渴了（谁 `@Autowired`）就直接去接水（拿同一个对象）。
- **Prototype（多例）**：就像你给员工发的**一次性纸杯**。每次有人要杯子（每次请求该 Bean），行政（Spring）就现去仓库拿一个全新的给他。每个人拿到的都是独立的，互不影响。

### 怎么用？（代码实操）

**默认情况（单例）：**

```java
@Service
// 等价于 @Scope("singleton")
public class EmployeeServiceImpl implements EmployeeService { ... }
```

无论你在项目里 `@Autowired` 多少次 `EmployeeService`，它们的内存地址（引用）完全一样。

**修改为多例：**

```java
@Service
@Scope("prototype") // 或者 @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class OrderServiceImpl implements OrderService { ... }
```

**🔥 面试高频：单例 Bean 存在线程安全问题吗？**

如何判断有无状态：只需判断这个对象里面是否保存数据

- 如果单例 Bean 是**无状态（Stateless**）的（比如你的 `EmployeeServiceImpl`，里面只有方法，没有可修改的成员变量），项目在启动时，可能有多个线程同时访问这个bean对象，由于没有额外保存其他数据，不存在多线程共享数据问题，那它是**绝对线程安全**的。项目开发时controller，service，dao都是属于无状态的bean，单例线程更加节约资源，性能也更高。
- 如果单例 Bean 是**有状态（Stateful**）的（比如里面写了一个 `private int count = 0;`，然后有自增操作），内部会保存状态信息，多个线程同时操作该bean时，可能会出现数据不一致问题，那它是**线程不安全**的！所以设置为多例，这样每次启动项目后，count记录的就是与本次操作相关的数据
- **解决方案**：
    1. 尽量避免在单例 Bean 中写可变的成员变量。
    2. 如果必须有，使用 `ThreadLocal` 把变量隔离。
    3. 实在不行，把它改成 `@Scope("prototype")`（但会导致频繁创建对象，不推荐）。

### 底层原理（作用域篇：Spring 在背后干了什么？）

Spring 底层是通过一个巨大的 `ConcurrentHashMap` 来管理单例 Bean 的，这个 Map 通常被称为“单例池”（`singletonObjects`）。

1. **Singleton 的底层逻辑**：
    当你执行 `context.getBean("employeeService")` 时：
    - Spring 先去 `singletonObjects` 这个 Map 里找 Key。
    - 如果找到了，直接返回缓存里的对象。
    - 如果没找到，走完整的 Bean 生命周期创建对象，然后 `map.put("employeeService", bean)` 存入缓存，再返回。
    - _注意：这就是为什么单例 Bean 能避免重复创建，极大地节省了内存。_
2. **Prototype 的底层逻辑**：
    当你执行 `context.getBean("orderService")` 时：
    - Spring **跳过**去缓存查找的步骤（或者明确知道要跳过）。
    - 直接走完整的 Bean 生命周期创建对象。
    - **创建完毕后，不往 `singletonObjects` 这个 Map 里放！**
    - 直接返回给你。
    - _注意：这也是为什么 Spring 不负责销毁 Prototype 的原因，因为它根本没记录这个对象。_

### 实战最佳实践

- **绝大多数情况**：使用默认的 **Singleton**。Spring 的 Controller、Service、Mapper 全都是单例，因为它们是“无状态”的，只负责处理逻辑，不保存业务数据。这样性能最好。
- **极少情况**：使用 **Prototype**。例如，某个对象内部需要携带大量属于当前请求的数据（比如一个封装了复杂状态的计算器对象），并且该对象非线程安全，此时才用多例。

单例的bean默认在项目启动时创建的。如果想要延迟创建，可以加一个@Lazy延迟创建---延迟到第一次使用时才会创建
