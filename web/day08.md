# Day 08 笔记：AOP 进阶、ThreadLocal、SpringBoot 配置与 Bean

---

## 一、AOP 通知顺序

当多个切面的切点都匹配到目标方法时，目标方法运行，**多个通知方法都会被执行**。

### 默认执行顺序（类名字母排序）

不同切面类中，默认按**类名字母排序**（后进先出）：

- 目标方法**前**的通知方法：字母排名靠前的**先执行**
- 目标方法**后**的通知方法：字母排名靠前的**后执行**

**示例：** aop 包下定义 `MyAspect1`、`MyAspect3` 两个类：

| 通知类型 | 执行顺序 |
|---------|---------|
| 前置通知（`@Before`） | `MyAspect1` → `MyAspect3` |
| 后置通知（`@After`） | `MyAspect3` → `MyAspect1` |

> **速记：** 按类名字母排，前置正着跑，后置反着跑。

### 用 `@Order` 控制顺序

业务中方法名都和业务挂钩，`1`、`2`、`3` 这种字母排序不太适合。可在切面类上加 `@Order(数字)` 控制顺序：

- 目标方法**前**的通知：数字**小的先执行**
- 目标方法**后**的通知：数字**小的后执行**

> **速记：** `@Order` 数字越小优先级越高，前置小先跑，后置小后跑。

---

## 二、切入点表达式

### 介绍与作用

**介绍：** 描述切入点方法的一种表达式。

**作用：** 决定项目中哪些方法需要加入通知。

### 常见形式

1. **`execution(...)`**：根据方法签名匹配
2. **`@annotation(...)`**：根据注解匹配
3. 根据业务需要，可以使用 **且（`&&`）、或（`||`）、非（`!`）** 组合出更复杂的切入点表达式

> **速记：** 两大基础 `execution` 和 `@annotation`，用 `&&` `||` `!` 自由组合。

---

## 三、切入点表达式 — `execution`

### 基本用法

```java
@Before("execution(public void com.itheima.service.impl.DeptServiceImpl.delete(java.lang.Integer))")
public void before(JoinPoint joinPoint) { ... }
```

### 语法

```
execution(访问修饰符? 返回值 包名.类名.? 方法名(方法参数) throws 异常?)
```

其中带 `?` 的部分可省略：

| 部分 | 说明 |
|------|------|
| 访问修饰符 | 可省略（如 `public`、`protected`） |
| 包名.类名 | 可省略（**不建议**） |
| throws 异常 | 可省略（注意是方法**声明**抛出的异常，不是实际抛出的异常） |

### 通配符

| 通配符 | 说明 | 示例 |
|--------|------|------|
| `*` | 单个任意符号，可通配任意返回值、包名、类名、方法名、**单个**任意类型参数，也可通配包/类/方法名的一部分 | `execution(* com.*.service.*.update*(*))` |
| `..` | 多个连续任意符号，通配**任意层级**包，或**任意类型、任意个数**的参数 | `execution(* com.itheima..DeptService.*(..))` |

> **速记：** 星号 `*` 匹配单个，双点 `..` 匹配任意层级和任意个数参数。

---

## 四、综合示例 — 一个切面中多种通知共存

### 业务代码

```java
@Service
public class DeleteServiceImpl implements DeleteService {
    @Override
    public void delete(Long id) {
        System.out.println("执行删除：" + id);
    }
}
```

调用方：

```java
deleteService.delete(1L);
```

### 添加切面

```java
@Aspect
@Component
public class DeleteAspect {

    @Pointcut("execution(* com.itheima.service.impl.DeleteServiceImpl.delete(..))")
    public void deletePointcut() {}

    @Before("deletePointcut()")
    public void before() {
        System.out.println("【前置】权限校验通过");
    }

    @Around("deletePointcut()")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } finally {
            System.out.println("【环绕】耗时 " + (System.currentTimeMillis() - start) + "ms");
        }
    }

    @AfterReturning("deletePointcut()")
    public void afterReturning() {
        System.out.println("【返回】记录操作日志");
    }
}
```

### 输出结果

```text
【前置】权限校验通过
【环绕】开始
执行删除：1
【返回】记录操作日志
【环绕】耗时 3ms
```

### 小结

> `DeleteServiceImpl.delete` 里只有一行 `System.out.println`，但实际执行时多出了权限校验、耗时统计、操作日志。这就是 **AOP 的价值**——业务代码保持干净，横切逻辑通过切点自动织入。

> **速记：** 业务方法保持干净，横切逻辑用切点自动织入。

---

## 五、`@annotation` 表达式总结

### 作用

匹配**方法上带指定注解**的方法，常用于自定义注解 + AOP 增强。

### 前提

自定义注解必须：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)   // 必须 RUNTIME
public @interface Loggable {
    String value() default "";
}
```

### 语法

```
@annotation(注解全限定名)
@annotation(变量名)
```

### 基本用法

```java
@Pointcut("@annotation(com.example.annotation.Loggable)")
public void logPointcut() {}

@Before("logPointcut()")
public void before(JoinPoint jp) {}
```

### 绑定注解属性

```java
@Before("@annotation(loggable)")
public void before(JoinPoint jp, Loggable loggable) {
    System.out.println(loggable.value());
}
```

> 变量名要一致，否则用 `@argNames` 指定。

### 组合限定

```java
@Pointcut("execution(* com.example.service..*(..)) && @annotation(loggable)")
public void pc(Loggable loggable) {}
```

### 手动获取注解

```java
MethodSignature sig = (MethodSignature) pjp.getSignature();
Loggable loggable = sig.getMethod().getAnnotation(Loggable.class);
```

### 与相似表达式区别

| 表达式 | 匹配位置 |
|--------|---------|
| `@annotation` | 方法上的注解 |
| `@within` | 类上的注解 |
| `@target` | 目标对象类上的注解 |
| `@args` | 方法参数类型上的注解 |

### 常见坑

- 注解必须是 `RUNTIME`
- 必须标在方法上
- 必须写全限定名
- 绑定变量名要一致
- 自调用失效
- `private` / `final` / `static` 方法不生效
- 非 Spring Bean 不生效

> **速记：** `@annotation` 匹配方法注解，RUNTIME + 方法上 + 全限定名，坑不少记得清。

---

## 六、ThreadLocal

### 概念

**线程的局部变量**，为每一个线程提供**单独一份存储空间**，具有线程隔离效果，不同线程之间不会相互干扰。

### 应用场景

在**同一线程 / 同一请求**中进行数据共享。

> **速记：** 线程隔离存一份，同线程内共享数据。

---

## 七、SpringBoot 配置优先级

从**高到低**：

| 优先级 | 配置方式 |
|--------|---------|
| 1（最高） | 命令行参数（`--xxx=xxx`） |
| 2 | java 系统属性（`-Dxxx=xxx`） |
| 3 | `.properties` |
| 4 | `.yml` / `.yaml`（项目开发通常使用） |

> **速记：** 命令行 > 系统属性 > properties > yml，同级 yml 和 properties 中 properties 更高。

---

## 八、Bean 作用域

### 五种作用域总述

| 作用域 | 说明 | 生效环境 |
|--------|------|---------|
| `singleton`（默认） | 容器内同名 bean **只有 1 个实例**，容器启动时实例化 | 所有环境 |
| `prototype` | 每次获取/使用都**新建一个实例** | 所有环境 |
| `request` | **每次 HTTP 请求**创建一个新实例，请求结束销毁 | 仅 Web |
| `session` | **每个用户会话**创建一个新实例，会话过期销毁 | 仅 Web |
| `application` | **整个 web 应用**范围内只有 1 个实例，等同 `ServletContext` | 仅 Web |

### 延迟加载

默认 `singleton` 的 bean 在容器启动时被创建，可使用 `@Lazy` 注解**延迟初始化**（延迟到第一次使用时才创建）。

> **速记：** 五种作用域两普通三 Web，单例默认启动建，`@Lazy` 懒加载到首次用。

---

## 九、面试题

### 面试题 1：Spring 容器的 bean 是单例还是多例？单例 bean 什么时候实例化？

1. Bean 默认**单例（singleton）**，可通过 `@Scope("prototype")` 改为多例
2. 单例 Bean 默认实例化时机：**项目启动、Spring 容器创建时**就实例化，提前创建好对象存入容器
3. 延迟加载：添加 `@Lazy` 注解，单例 Bean 不会在启动时创建，**第一次获取使用时才实例化**

> **速记：** 默认单例启动建，`@Lazy` 延迟到首次用。

### 面试题 2：Spring 容器的 bean 是线程安全的吗？

Spring 本身**不保证 Bean 的线程安全**，是否安全取决于 Bean 有没有状态、作用域：

| 类型 | 线程安全性 | 说明 |
|------|----------|------|
| **无状态 Bean** | 线程安全 | 没有成员变量，不保存数据（如 Controller、Service、Mapper），多线程共用同一对象不会互相影响 |
| **有状态 Bean** | 线程不安全 | 单例只有一份，多线程同时读写成员变量会数据覆盖、错乱 |
| **prototype 多例** | 对象层面安全 | 每次获取新建对象；但对象内共享可变数据仍有线程安全问题 |

> **速记：** Spring 不负责线程安全，无状态单例安全，有状态单例不安全。

---

## 十、第三方 Bean — `@Bean` 注解

### 场景

如果要管理的 bean 对象**来自第三方**（不是自定义的），无法用 `@Component` 及衍生注解声明，就需要用 `@Bean` 注解。

### 注意事项

| 要点 | 说明 |
|------|------|
| 依赖注入 | 第三方 bean 需要依赖其它 bean，直接在 `@Bean` 方法中设置**形参**，容器根据类型自动装配 |
| 默认名称 | 通过 `@Bean` 的 `name` 或 `value` 属性可声明 bean 名称；**不指定则默认名称为方法名** |

> **速记：** 第三方类用 `@Bean`，形参自动装配，名称默认取方法名。

---

## 📌 今日重点

| # | 知识点 | 一句话记住 |
|---|--------|-----------|
| 1 | 通知执行顺序 | 默认类名字母排序，前置正着跑、后置反着跑；`@Order` 数字越小优先级越高 |
| 2 | 切入点表达式 | `execution` 匹配方法签名，`@annotation` 匹配方法注解，`&&` `\|\|` `!` 组合 |
| 3 | `execution` 通配符 | `*` 单个任意，`..` 任意层级包、任意个数参数 |
| 4 | `@annotation` 常见坑 | 注解必须 RUNTIME、标在方法上、写全限定名；自调用/private/final/static 失效 |
| 5 | ThreadLocal | 线程隔离变量，同一线程/同一请求内共享数据 |
| 6 | 配置优先级 | 命令行 > 系统属性 > properties > yml |
| 7 | Bean 五种作用域 | singleton/prototype 通用，request/session/application 仅 Web；单例默认启动时创建，`@Lazy` 懒加载 |
| 8 | 单例 bean 线程安全 | Spring 不保证线程安全：无状态安全，有状态不安全 |
| 9 | 第三方 Bean | 用 `@Bean` 声明，形参自动装配，名称默认取方法名 |

