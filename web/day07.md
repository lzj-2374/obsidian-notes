# Day 07 笔记：JWT 令牌、过滤器、拦截器与 SpringAOP

---

## 一、JWT 令牌登录校验

### 流程

用户登录成功后，系统自动下发 **JWT 令牌**；后续每次请求都在**请求头 header** 中携带：

| 请求头名称 | 值 |
|-----------|-----|
| `token` | 登录时下发的 JWT 令牌 |

- 检测到用户**未登录**（无令牌/令牌无效）→ 直接响应 **401 状态码**

> **速记：** 登录发 JWT，请求头带 token，没令牌给 401。

---

## 二、过滤器 Filter

### 概念

过滤器可以把对资源的请求拦截下来，实现一些特殊功能。一般完成**通用操作**：登录校验、统一编码处理、敏感字符处理等。

### 开发步骤

**1. 定义：** 实现 `Filter` 接口（`init`、`doFilter`、`destroy` 三个方法）

**2. 配置：**

```java
@WebServlet(urlPatterns = "/*")   // 拦截所有请求
@ServletComponentScan            // 开启 SpringBoot 对 Servlet 组件的支持，写在启动类上
```

### 完整代码示例

```java
@WebServlet(urlPatterns = "/*")  // 拦截所有请求
@Slf4j
public class DemoFilter implements Filter {

    // 初始化方法：Web 服务器启动时执行，只执行一次
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        log.info("init 初始化方法。。。");
    }

    // 拦截到请求后执行，执行多次
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
                         FilterChain filterChain) throws IOException, ServletException {
        log.info("doFilter 拦截到请求了。。。");
        // 放行
        filterChain.doFilter(servletRequest, servletResponse);
    }

    // 销毁方法：Web 服务器关闭时执行，只执行一次
    @Override
    public void destroy() {
        log.info("destroy 销毁方法。。。");
    }
}
```

### 注意事项

| 要点 | 说明 |
|------|------|
| **不放行就访问不到资源** | 过滤器中必须执行 `filterChain.doFilter(request, response)` 放行 |
| **登录/注册不校验** | 登录、注册请求被拦截后不需要校验令牌，直接放行 |
| **有令牌且校验通过才放行** | 否则返回未登录错误（401） |
| **放行后还会回到 Filter** | 资源访问结束后回到 Filter，执行放行后的逻辑 |

### 令牌校验 Filter 流程

```
1. 获取请求 URL
2. 判断 URL 是否包含 "login" → 包含说明是登录操作，直接放行
3. 获取请求头中的令牌 (token)
4. 判断令牌是否存在 → 不存在，响应 401
5. 解析 token → 解析失败，响应 401
6. 放行
```

> **速记：** 三步放行：登录不校验、无令牌 401、解析失败 401，通过才放行。

---

## 三、拦截器 Interceptor

### 概念

**Spring 框架提供**的动态拦截方法调用机制，类似过滤器，主要用来动态拦截**控制器方法**的执行。

**作用：** 在指定的方法调用前后，根据业务需要执行预先设定的代码。

### 开发步骤

**1. 定义拦截器** — 实现 `HandlerInterceptor` 接口，三个方法：

| 方法 | 执行时机 |
|------|---------|
| `preHandle` | 请求处理之前 |
| `postHandle` | 请求处理之后（视图渲染之前） |
| `afterCompletion` | 整个请求完成之后 |

**2. 配置** — 定义配置类实现 `WebMvcConfigurer` 接口，注册拦截器：

```java
@Override
public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(demoInterceptor)
            .addPathPatterns("/**")        // 拦截所有路径
            .excludePathPatterns("/login"); // 放行登录接口
}
```

> **速记：** 实现 HandlerInterceptor 三方法，WebMvcConfigurer 里注册，add 拦截 exclude 放行。

---

## 四、SpringMVC 拦截器路径匹配

### 核心方法

```java
registry.addInterceptor(拦截器对象)
       .addPathPatterns(拦截路径)      // 配置需要拦截的路径
       .excludePathPatterns(放行路径);  // 配置不需要拦截、直接放行的路径
```

### 路径匹配规则（重点）

| 模式 | 含义 | 匹配示例 | 不匹配示例 |
|------|------|---------|-----------|
| `/*` | 匹配**一级路径** | `/depts`、`/login` | `/depts/1` |
| `/**` | 匹配**任意多级路径**（最常用） | `/depts`、`/depts/1`、`/depts/1/2` | — |
| `/depts/*` | 匹配 `/depts` 下**一级子路径** | `/depts/1` | `/depts`、`/depts/1/2` |
| `/depts/**` | 匹配 `/depts` 下**所有任意级子路径** | `/depts`、`/depts/1`、`/depts/1/2` | `/emps/1` |

### 示例代码含义

```java
registry.addInterceptor(demoInterceptor).addPathPatterns("/**").excludePathPatterns("/login");
```

拦截**所有请求**，唯独放行 `/login` 登录接口。

> **速记：** 星号一级 `*`，双星号任意级 `**`；带前缀 `/depts/**` 才限定在该前缀下。

---

## 五、Filter vs Interceptor 对比（面试常问）

### 执行顺序

```
Filter（过滤器） → Interceptor（拦截器） → Controller
```

### 核心区别 — 作用范围

| 组件 | 归属 | 作用范围 |
|------|------|---------|
| **Filter 过滤器** | Servlet 规范，是 **Tomcat 等 Web 服务器**的组件 | 拦截**所有**到达服务器的请求（静态资源、JSP、Servlet、Spring 接口全经过），作用在 **Spring 框架之外** |
| **Interceptor 拦截器** | **SpringMVC** 框架自己的组件 | 只拦截**进入 SpringMVC 的请求**；没进 Spring 的（如静态资源被 Tomcat 处理）不生效 |

### 补充考点

| 考点 | Filter | Interceptor |
|------|--------|-------------|
| **触发时机** | Servlet 执行前 | DispatcherServlet 分发后、Controller 执行前 |
| **依赖** | 不依赖 Spring 容器 | 由 Spring 管理，可直接注入 Spring Bean |

> **速记：** 先 Filter 后 Interceptor；Filter 拦所有进服务器的，Interceptor 只拦进 Spring 的。

---

## 六、SpringAOP

### 概念

**AOP（Aspect Oriented Programming，面向切面编程）**，可简单理解为**面向特定方法编程**。

**场景：** 部分业务方法运行较慢，需要统计每一个业务方法的运行耗时。

**优势：**
1. 减少重复代码
2. 代码无侵入
3. 提高开发效率
4. 维护方便

> 提示：AOP 是一种思想，Spring 框架中对这种思想的实现就是 **SpringAOP**。

### 导入依赖

**SpringBoot 3：**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

**SpringBoot 4：**
```xml
<!-- AOP 依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aspectj</artifactId>
</dependency>
```

> **速记：** Boot3 用 starter-aop，Boot4 换 starter-aspectj。

### AOP 入门 — 统计业务层方法耗时

步骤：获取方法开始时间 → 运行原始方法 → 获取结束时间，计算执行耗时。

**应用场景：** 记录操作日志、事务管理、权限控制等。

---

## 七、AOP 核心概念

### 五大核心术语

| 术语 | 英文 | 说明 |
|------|------|------|
| **连接点** | JoinPoint | 可以被 AOP 控制的方法（暗含方法执行时的相关信息） |
| **通知** | Advice | 那些重复的逻辑，即共性功能（最终体现为一个方法） |
| **切入点** | PointCut | 匹配连接点的条件，通知仅会在切入点方法执行时被应用 |
| **切面** | Aspect | 描述通知与切入点的对应关系（通知 + 切入点） |
| **目标对象** | Target | 通知所应用的对象 |

> 关键关系：**切入点一定是连接点，但连接点不一定是切入点。**

> **速记：** 连接点是所有方法，切入点是筛出来的；通知 + 切入点 = 切面。

### 五种通知类型

| 注解 | 类型 | 说明 |
|------|------|------|
| `@Around` | 环绕通知 | 目标方法前、后都执行 |
| `@Before` | 前置通知 | 目标方法前执行 |
| `@After` | 后置通知 | 目标方法后执行，**无论是否有异常都会执行** |
| `@AfterReturning` | 返回后通知 | 目标方法后执行，**有异常不会执行** |
| `@AfterThrowing` | 异常后通知 | 发生异常后执行 |

### 注意事项

| 要点 | 说明 |
|------|------|
| `@Around` 需手动调用原始方法 | 必须调用 `ProceedingJoinPoint.proceed()`，其他通知不需要 |
| `@Around` 返回值类型 | 必须指定为 `Object`，用来接收原始方法的返回值 |

> **速记：** 环绕通知手动 proceed 跑原方法，返回类型写 Object。

---

## 📌 今日重点

| # | 知识点 | 一句话记住 |
|---|--------|-----------|
| 1 | JWT 校验 | 请求头带 token，无令牌给 401 |
| 2 | 令牌校验 Filter | 登录放行、无令牌 401、解析失败 401，通过才放行 |
| 3 | Filter 开发 | 实现 Filter 接口 + `@WebServlet` + 启动类 `@ServletComponentScan` |
| 4 | Interceptor 开发 | 实现 HandlerInterceptor + WebMvcConfigurer 注册 |
| 5 | 路径匹配 | `*` 一级，`**` 任意级 |
| 6 | Filter vs Interceptor | 先 Filter 后 Interceptor；Filter 拦所有，Interceptor 只拦进 Spring 的 |
| 7 | AOP 依赖 | Boot3 用 starter-aop，Boot4 用 starter-aspectj |
| 8 | 五通知类型 | Around/Before/After/AfterReturning/AfterThrowing |
| 9 | `@Around` 注意 | 手动 `proceed()` 执行原方法，返回类型 `Object` |
