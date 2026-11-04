# Day 06 笔记：登录功能与会话技术

---

## 一、登录功能本质

登录 = **一次查询**：根据用户名和密码查询员工信息，能查到就登录成功。

> **速记：** 登录就是一次按账号密码的查询，查得到就放行。

---

## 二、登录校验 — 为什么需要"登录标记"？

### 问题：HTTP 协议是无状态的

登录是浏览器 → Web 服务器发请求，服务器再响应的过程。但 **HTTP 是无状态协议**：后一次请求不会携带前一次请求的信息。

所以服务器无法自动知道"这个用户刚刚登录过"。

### 解决方案：登录标记 + 统一拦截

1. 登录成功时，**存储一个登录标记**，标识该员工已登录
2. 后续请求中，**统一拦截检查**这个标记：
   - **有标记** → 放行，允许访问功能接口
   - **无标记** → 拦截，响应错误结果，让前端跳转到登录页

> **速记：** HTTP 无状态记不住人，登录成功存标记，统一拦截查标记。

---

## 三、核心概念

### 登录标记 → 会话技术（Session）

用户登录成功后，**后续请求都能携带这个标记**来证明身份，这就是"会话技术"。

### 统一拦截 → 过滤器 / 拦截器

| 组件 | 提供方 | 说明 |
|------|--------|------|
| **Filter（过滤器）** | Java 提供 | Servlet 规范，作用于请求/响应的原始数据 |
| **Interceptor（拦截器）** | Spring 提供 | Spring MVC 层，作用于 Controller 方法前后 |

> **速记：** 标记找会话，拦截靠 Filter / Interceptor；Java 给 Filter，Spring 给 Interceptor。

---

## 四、会话技术详解

### 什么是会话（Session）

用户打开浏览器访问 Web 服务器资源 → **会话建立** → 直到一方断开连接 → 会话结束。一次会话中可以包含**多次请求和响应**。

### 什么是会话跟踪（Session Tracking）

一种维护浏览器状态的方法：服务器需要识别**多次请求是否来自同一浏览器**，以便在同一次会话的多次请求间共享数据。

### 三大会话跟踪方案

| 方案 | 类型 | 说明 |
|------|------|------|
| **Cookie** | 客户端 | 服务端下发，浏览器保存并自动携带 |
| **Session** | 服务端 | 服务端存储用户状态，配合 Cookie 识别 |
| **令牌（Token）** | 无状态 | 登录成功签发 Token，后续请求携带 |

> **速记：** 会话跟踪三方案，Cookie 在客户端，Session 在服务端，Token 无状态。

---

## 五、Cookie 知识点

### 工作流程

```
① 服务端 → 响应头 Set-Cookie: xxx
② 浏览器 → 自动保存 Cookie
③ 后续每次请求 → 浏览器自动携带 Cookie
④ 后端拦截器 → 读取 Cookie 完成身份校验
```

**关键点：** Cookie 是**服务端下发、浏览器保存、浏览器自动携带**，全程无需手动处理请求头。

### 优点

- HTTP 原生自带技术
- 浏览器原生支持，**不用手动处理请求头**

### 缺点

| 缺点 | 说明 |
|------|------|
| 移动端不适用 | APP 无法使用 Cookie |
| 可被禁用 | 用户可以手动禁用 Cookie，导致功能失效 |
| 安全隐患 | 存在 CSRF 等安全风险 |
| **不能跨域** | 不同域之间无法共享 Cookie |

> **速记：** Cookie 自动携带省心，但移动端不能用、可禁用、不能跨域。

---

# Session 知识点

1. 原理：登录成功后，服务端**创建 Session 对象（存在服务器内存）**，生成唯一标识`JSESSIONID`；通过`Set-Cookie`响应头把 JSESSIONID 发给浏览器保存。
2. 后续请求：浏览器自动携带 Cookie（`JSESSIONID`），服务端拦截器拿到这个 id，去服务器找到对应的 Session，完成身份校验。
3. 本质：**Session 是保存在服务端，依赖 Cookie 传递 JSESSIONID**。
4. 优缺点 优点：用户信息存在服务器，比 Cookie 安全；
   缺点：

- 底层依赖 Cookie，Cookie 有的问题它基本都有（移动端 APP 不好用、不能跨域）
- 服务器集群环境下，多台服务器需要做 Session 共享
- 服务器重启，内存里的 Session 数据就丢失

```
// 存值接口
@GetMapping("/s1")
public Result session1(HttpSession session){
    log.info("HttpSession-s1: {}", session.hashCode());
    session.setAttribute("loginUser", "tom"); //往session中存储数据
    return Result.success();
}

// 取值接口
@GetMapping("/s2")
public Result session2(HttpSession session){
    log.info("HttpSession-s2: {}", session.hashCode());
    Object loginUser = session.getAttribute("loginUser"); //从session中获取数据
    log.info("loginUser: {}", loginUser);
    return Result.success(loginUser);
}
```

## 核心知识点

1. **HttpSession 注入** SpringBoot 会自动注入`HttpSession`对象，不需要自己 new。底层会读取请求里的`JSESSIONID`，找到对应的服务端 Session。
2. **存数据：`session.setAttribute("key",value)`**

- 作用：在服务端的 session 对象中保存键值对数据
- 示例：`session.setAttribute("loginUser", "tom")`，key 为`loginUser`，值是 tom。

3. **取数据：`session.getAttribute("key")`**

- 根据 key 读取 session 里面保存的数据，**返回值是 Object 类型**，需要时自行强转。
- 如果 key 不存在，返回`null`。

4. **同一会话判断（hashCode）** 同一个浏览器会话访问 s1、s2，两次拿到的`session.hashCode()`值**相同**，代表是同一个 Session 对象。 原理：浏览器携带同一个 JSESSIONID，服务器匹配到同一个 Session。
5. 执行流程 ①访问 `/s1`：创建 Session，存入`loginUser=tom`；响应头带回 Set-Cookie，浏览器保存 JSESSIONID ②访问 `/s2`：请求自动带上 Cookie 里的 JSESSIONID；服务器根据 id 拿到同一个 session，读取里面保存的数据

# Token（令牌）知识点

1. **执行流程** 登录成功，后端生成 token 令牌返回给前端；前端保存 token，**后续每次请求，在请求头手动带上 token**；后端统一拦截器校验 token，校验通过允许访问接口。

> 和 Cookie 不一样：不是浏览器自动携带，前端代码主动放到请求头。

2. **优点**

- 同时支持 PC 网页端、移动端 APP
- 天然适配服务器集群，不存在 session 共享问题
- 用户信息存放在 token 里，**不占用服务器内存，减轻存储压力**

3. **缺点**

- 需要自己编码实现 token 生成、解析、校验逻辑
- token 一旦签发，有效期内无法直接作废（JWT 典型问题）

2. 对比小结 Cookie：数据在浏览器，自动携带；移动端、跨域不行 Session：数据在服务端，依赖 Cookie；集群麻烦 Token：令牌交给前端保存，请求头传递；全端支持，适合集群

# JWT 令牌知识点

1. **全称**：JSON Web Token
2. **作用**：简洁、自包含格式，以 JSON 安全在通信双方传递信息。
3. **JWT 由 3 部分组成，用`.`隔开**

- Header（头）：记录令牌类型、签名算法。例：`{"alg":"HS256","type":"JWT"}`
- Payload（有效载荷）：存放自定义信息、默认信息（id、username 等）。**注意：仅 Base64 编码，不是加密，不能放敏感数据**
- Signature（签名）：防止 JWT 被篡改。把 Header、Payload 拼接，加上秘钥，使用指定算法算出签名。

> 完整格式：`Header.Base64编码.Payload.Base64编码.签名`

4. **Base64**：编码方式，用 A-Z a-z 0-9 + / 表示二进制数据；末尾的`=`是**占位符**，用于补齐长度，不属于有效内容。
5. 安全要点

- Header 和 Payload 只是编码，可以直接解码查看内容
- 签名用来校验：一旦 header 或 payload 内容被修改，签名校验就失败，判定令牌无效

# JWT 生成与解析知识点

1. JWT 三段结构：`Header.Base64编码.Payload.Base64编码.数字签名`，三段之间用`.`分割
    - Header：存放算法、令牌类型
    - Payload：存放自定义载荷数据（如 name、iat 签发时间）
    - 数字签名：由 header、payload + 秘钥，使用指定算法计算，用来校验内容有没有被篡改
2. Java 操作 JWT 使用`jjwt`工具包，第一步需要在 pom.xml 引入 jjwt 依赖
3. 核心工具类：`Jwts`，调用它里面的方法**生成 JWT 令牌、解析 JWT 令牌**
4. 生成 JWT 逻辑：组装 header、payload，设置秘钥，自动生成签名，拼接成完整 JWT 字符串返回 Jwts.bulider()...
5. 解析 JWT 逻辑：传入 JWT 字符串和秘钥，jjwt 内部自动校验签名；签名校验失败直接抛出异常 Jwts.parser()...
解析（校验）jwt令牌报错：jwt令牌被篡改或者过期了
注意事项：JWT校验时使用的签名密钥，必须和生成JWT令牌时使用的密钥是配套的
## 📌 今日重点

| # | 知识点 | 一句话记住 |
|---|--------|-----------|
| 1 | 登录本质 | 按用户名密码查询员工，查得到即成功 |
| 2 | 为什么需要登录标记 | HTTP 无状态，后续请求不携带上次信息，需存标记证明身份 |
| 3 | 统一拦截 | 有标记放行，无标记拦截跳登录页 |
| 4 | Filter vs Interceptor | Java 提供 Filter，Spring 提供 Interceptor |
| 5 | 会话跟踪三方案 | Cookie（客户端）、Session（服务端）、Token（无状态） |
| 6 | Cookie 流程 | 服务端 Set-Cookie → 浏览器保存 → 自动携带 → 拦截器校验 |
| 7 | Cookie 缺点 | 移动端不能用、可禁用、有安全隐患、不能跨域 |

