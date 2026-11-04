# Day01 笔记

> [!info] 项目
> 苍穹外卖 · 第 1 天
>
> 涉及技术：Spring Boot、Nginx、Knife4j（Swagger）、MyBatis、JWT、ThreadLocal

- **tags**: #苍穹外卖 #SpringBoot #Nginx #Knife4j #ThreadLocal

---

## 1. 对象分类（Entity / DTO / VO / POJO）

1. **Entity（实体类）**：对应数据库表，属性与表字段一一映射，MyBatis/JPA 使用，用来接收数据库查询出来的数据。
   > 属于 POJO 的一种。
2. **DTO（Data Transfer Object，数据传输对象）**：用于**层与层之间传递数据**（前端 → Controller → Service）。
   作用：接收前端提交参数，过滤多余字段，不直接把 Entity 暴露给前端。
3. **VO（View Object，视图对象）**：返回给**前端展示**的对象。
   作用：组装后端数据，按需返回页面需要的字段，隐藏敏感字段。
4. **POJO（Plain Old Java Object，普通 Java 对象）**：泛指简单 Java 类，**只有属性 + getter/setter**，不继承特定类、不实现特定接口。
   > Entity、DTO、VO 本质都属于 POJO。

### 1.1 数据流转流程（面试重点）

```
前端 → DTO → Controller → Service → Entity → Mapper/数据库
数据库 → Entity → Service → VO → Controller → 前端
```

---

## 2. Nginx

Nginx 反向代理：将前端发送的动态请求由 Nginx 转发到后端服务器。

### 2.1 反向代理的好处

- 提高访问速度
- 进行负载均衡（负载均衡：把大量请求按指定方式均衡地分配给集群中的每台服务器）
- 保证后端服务安全

### 2.2 反向代理配置方式

```nginx
server{
    listen 80;
    server_name localhost;

    location /api/ {
        proxy_pass http://localhost:8080/admin/;   # 反向代理
    }
}
```

示例：`http://localhost/api/employee/login` → Nginx → `http://localhost:8080/admin/employee/login`

### 2.3 负载均衡配置方式

```nginx
upstream webservers{
    server 192.168.100.128:8080;
    server 192.168.100.129:8080;
}

server{
    listen 80;
    server_name localhost;

    location /api/ {
        proxy_pass http://webservers/admin/;   # 负载均衡
    }
}
```

两台 Tomcat 服务器，`webservers` 名称要保持一致。

**Nginx 负载均衡策略**：

| 名称 | 说明 |
|---|---|
| 轮询 | 默认方式 |
| weight | 权重方式，默认为 1，权重越高，被分配的客户端请求就越多 |
| ip_hash | 依据 IP 分配，每个访客可固定访问同一个后端服务 |
| least_conn | 依据最少连接方式，把请求优先分配给连接数少的后端服务 |
| url_hash | 依据 URL 分配，相同的 URL 会被分配到同一个后端服务 |
| fair | 依据响应时间方式，响应时间短的服务会被优先分配 |

> [!tip] 补充拓展（面试/实战注意点）
> - **前三种（轮询、weight、ip_hash）** 是 Nginx **原生自带**的策略（least_conn 也属于原生）。
> - **url_hash 和 fair** 通常需要安装**第三方模块**（如 `nginx-upstream-fair`）才能使用。
> - 如果是基于 URL 哈希，现在官方更推荐使用 `hash $request_uri consistent;` 语法。
> - 负载均衡配置通常写在 `upstream` 块中，如：`upstream webservers { server 192.168.1.1 weight=2; ... }`。

---

## 3. IDEA 小技巧

未完成的工作可以加一个 `//TODO xxx` 标签，在 IDEA 下方有个 TODO 按钮，点一下就能看到未完成的工作了。

---

## 4. 密码 MD5 加密

对前端传递过来的密码进行 MD5 加密处理：

```java
password = DigestUtils.md5DigestAsHex(password.getBytes());
```

---

## 5. Swagger / Knife4j 接口文档

使用 Swagger 按照规范定义接口及接口相关信息，即可生成接口文档以及在线接口调试页面。
Knife4j 是为 Java MVC 框架集成 Swagger 生成 API 文档的增强解决方案，直接引入依赖即可。

### 5.1 引入依赖

```xml
<dependency>
    <groupId>com.github.xiaoymin</groupId>
    <artifactId>knife4j-spring-boot-starter</artifactId>
    <version>3.0.2</version>
</dependency>
```

### 5.2 配置类

创建一个 Java 配置类来启用 Knife4j 并配置文档的基本信息（标题、描述、扫描的包路径等）：

```java
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Info;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class Knife4jConfiguration {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("SpringBoot整合Knife4j-api接口文档")
                        .description("本文档描述了SpringBoot如何整合Knife4j")
                        .version("1.0"));
    }
}
```

### 5.3 使用注解描述接口

在 Controller 和实体类上使用 Swagger/OpenAPI 注解，让生成的文档更清晰。

| 注解 | 作用 | 使用位置 |
|---|---|---|
| `@Tag(name = "用户管理接口")` | 描述 Controller 模块 | Controller 类上 |
| `@Operation(summary = "用户登录")` | 描述具体接口方法 | Controller 方法上 |
| `@Schema(description = "用户登录参数")` | 描述实体类或字段 | 实体类或字段上 |
| `@Parameter(description = "用户ID")` | 描述请求参数 | 方法参数上 |

> [!warning] 注意
> OpenAPI 2 和 OpenAPI 3 的注解名称不同：`@Api` → `@Tag`，`@ApiOperation` → `@Operation`，`@ApiModelProperty` → `@Schema`。

### 5.4 启动并访问

完成以上配置后，启动 Spring Boot 应用，在浏览器中访问：

**`http://localhost:8080/doc.html`**

即可看到 Knife4j 提供的增强版接口文档界面。在这里可以查看接口详情、在线调试、设置全局参数等。

**通过 Swagger 生成了接口文档，那么就不需要 YApi 了？**

1. YApi 是**设计阶段**使用的工具，用于管理和维护接口；
2. Swagger 是**开发阶段**使用的框架，帮助后端开发人员做后端的接口测试。

### 5.5 Swagger 2（Springfox）常用注解对照

**核心说明**：通过注解可以控制生成的接口文档，使接口文档拥有更好的可读性。

| Swagger 2（Springfox） | 说明 |
|---|---|
| `@Api` | 用在类上（如 Controller），表示对类的说明 |
| `@ApiModel` | 用在类上（如 entity、DTO、VO） |
| `@ApiModelProperty` | 用在属性上，描述属性信息 |
| `@ApiOperation` | 用在方法上（如 Controller 的方法），说明方法的用途、作用 |

Knife4j 4.x 及以上版本（基于 OpenAPI 3）注解发生改变，对应关系如下：

- `@Api` ➡️ 替换为 `@Tag`
- `@ApiModel` ➡️ 替换为 `@Schema`
- `@ApiModelProperty` ➡️ 替换为 `@Schema`
- `@ApiOperation` ➡️ 替换为 `@Operation`

---

## 6. 对象属性拷贝

```java
BeanUtils.copyProperties(source, target);
```

将 `source` 里的属性拷贝到 `target` 里；注意，`source` 里面的属性名要和 `target` 里的一样。

---

## 7. ⭐重点：ThreadLocal + JWT 获取当前登录员工 ID

新增员工时需要记录创建人和修改人 ID。登录时，拦截器会校验生成的 JWT 令牌；校验通过后，把登录的员工 ID 记录到**线程局部变量**中，Service 层即可取出。

### 7.1 拦截器中校验 JWT 并取出 empId

```java
// 1、从请求头中获取令牌
String token = request.getHeader(jwtProperties.getAdminTokenName());

// 2、校验令牌
try {
    Claims claims = JwtUtil.parseJWT(jwtProperties.getAdminSecretKey(), token);
    Long empId = Long.valueOf(claims.get(JwtClaimsConstant.EMP_ID).toString());
    // 3、校验通过，放行
    return true;
} catch (Exception ex) {
    // 4、校验不通过，响应 401 状态码
    response.setStatus(401);
    return false;
}
```

问题：记录的 empId 如何传递给 Service 层 `EmployeeServiceImpl` 的 save 方法？

### 7.2 ThreadLocal

ThreadLocal **并不是一个 Thread**，而是 Thread 的局部变量：为每个线程提供单独一份存储空间，具有线程隔离效果，只有在线程内才能获取到对应的值，线程外则不能访问。

ThreadLocal 常用方法：

```java
public void set(T value)   // 设置当前线程的线程局部变量的值
public T get()            // 返回当前线程所对应的线程局部变量的值
public void remove()      // 移除当前线程的线程局部变量
```

**原理说明**：客户端每发送一次请求，都会是一个单独的线程（比如点击"添加员工"的保存按钮这个请求，就是一个单独的线程）。在整个线程的生命周期内，都可以共享一份单独的存储空间。因此可以在拦截器处把当前用户 ID 存到存储空间里，当程序执行到 Service 层时，再从存储空间里取出 ID。

### 7.3 具体做法

在 `context` 包下定义 `BaseContext` 类，封装 ThreadLocal 的三个方法：

```java
public class BaseContext {

    public static ThreadLocal<Long> threadLocal = new ThreadLocal<>();

    public static void setCurrentId(Long id) {
        threadLocal.set(id);
    }

    public static Long getCurrentId() {
        return threadLocal.get();
    }

    public static void removeCurrentId() {
        threadLocal.remove();
    }
}
```

在拦截器 JWT 校验处使用这个工具类：

```java
// 2、校验令牌
try {
    log.info("jwt校验:{}", token);
    Claims claims = JwtUtil.parseJWT(jwtProperties.getAdminSecretKey(), token);
    Long empId = Long.valueOf(claims.get(JwtClaimsConstant.EMP_ID).toString());
    log.info("当前员工id: {}", empId);
    // 将当前员工id存到当前线程的存储空间
    BaseContext.setCurrentId(empId);
    // 3、校验通过，放行
    return true;
} catch (Exception ex) {
    // 4、校验不通过，响应 401 状态码
    response.setStatus(401);
    return false;
}
```

接下来就能在 Service 层的 save 方法里将这个 id 取出来：

```java
// 设置当前记录创建人和修改人id
employee.setCreateUser(BaseContext.getCurrentId());
employee.setUpdateUser(BaseContext.getCurrentId());
```

> [!tip] 要点回顾
> 拦截器（JWT 校验）→ `BaseContext.setCurrentId(empId)` → Service 层 `BaseContext.getCurrentId()` → 赋值给 Entity 的 createUser / updateUser。

---

## 8. 面试问题

> [!question] 🎤 面试题
> **问题**：苍穹外卖里，拦截器校验 JWT 后如何把"当前登录员工 ID"传递到 Service 层？为什么用 ThreadLocal 而不是层层传方法参数？

**参考答案**：

1. **传递方式**：拦截器解析 JWT 拿到 empId 后调用 `BaseContext.setCurrentId(empId)` 存入 `ThreadLocal`；Service 层再 `BaseContext.getCurrentId()` 取出，赋值给实体的 createUser / updateUser。
2. **为什么用 ThreadLocal**：
   - 一次 HTTP 请求由一个线程处理，从拦截器到 Service 都在**同一线程**内，ThreadLocal 为该线程提供独立存储空间，线程间互不干扰；
   - 相比每个方法都加参数传递，**无侵入**，不用改大量已有方法签名。
3. **注意事项**：请求结束（拦截器 afterCompletion / Filter）必须 `threadLocal.remove()`，否则线程池复用线程时会发生**内存泄漏**和**数据串线程**（拿到上一个请求的 empId）。
