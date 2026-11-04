# Day03 · 短信登录：从 Session 到 Redis 共享 Session

> **一句话概括**：登录功能的核心是"**服务端记住你是谁**"。单机时代用 `HttpSession` 就能做到，但集群下 Session 不共享，于是改成用 **Redis 存 token + 用户信息**，配合 **SpringMVC 拦截器 + ThreadLocal** 完成全局登录校验。

---

## 目录

- [[#一、项目导入与数据库表]]
- [[#二、登录方案整体演进]]
- [[#三、方案一：基于 Session 实现登录]]
- [[#四、SpringMVC 拦截器基础]]
- [[#五、Session 版登录拦截器]]
- [[#六、集群下的 Session 共享问题]]
- [[#七、方案二：基于 Redis 实现共享 Session 登录]]
- [[#八、面试考点]]
- [[#九、今日重点]]

---

## 一、项目导入与数据库表

数据库 `hmdp`，共 8 张表：

| 表名 | 说明 |
| --- | --- |
| `tb_user` | 用户表 |
| `tb_user_info` | 用户详细信息表 |
| `tb_shop` | 商户信息表 |
| `tb_shop_type` | 商户类型表 |
| `tb_blog` | 用户日记表（达人探店日记） |
| `tb_follow` | 用户关注表 |
| `tb_voucher` | 优惠券表 |
| `tb_voucher_order` | 优惠券订单表 |

---

## 二、登录方案整体演进

```
① 单体应用：HttpSession 存验证码和用户
        ↓ 问题：多台 Tomcat 不共享 session
② 集群部署：Session 复制 / 粘性会话 → 性能差、不彻底
        ↓
③ 最终方案：Redis 存 token + 用户信息（共享、内存、KV 结构）
```

**为什么是 Redis**：替代方案必须同时满足 **① 数据共享 ② 内存结构 ③ key-value 结构** —— Redis 刚好全中。

---

## 三、方案一：基于 Session 实现登录

### 3.1 发送短信验证码

**流程**：

```
开始 → 提交手机号 → 校验手机号（不符合则回退）→ 生成验证码
     → 保存验证码到 Session → 发送短信验证码 → 结束
```

**实现**：

```java
@Override
public void sendCode(String phone, HttpSession session) {
    // 1. 校验手机号（RegexUtils 是自己写的校验工具类）
    if (RegexUtils.isPhoneInvalid(phone)) {
        // 不符合，抛出业务异常（ResultCode 是自定义的错误码类）
        throw new BusinessException(ResultCode.PHONE_STYLE_ERROR);
    }

    // 2. 符合，生成 6 位验证码（hutool 的 RandomUtil）
    String code = RandomUtil.randomNumbers(6);

    // 3. 保存验证码到 session
    session.setAttribute("code", code);

    // TODO 发送验证码（真实项目接短信服务商）
}
```

---

### 3.2 短信验证码登录 / 注册

**流程**：

```
开始 → 提交手机号+验证码 → 校验验证码（不一致则回退）
     → 根据手机号查询用户 → 用户是否存在？
        不存在 → 创建新用户 → 保存到数据库
        存在   → 跳过
     → 保存用户到 Session → 结束
```

**实现**：

```java
/**
 * 登录功能
 */
@Override
public void login(LoginFormDTO loginForm, HttpSession session) {
    // 1. 校验手机号
    String phone = loginForm.getPhone();
    if (RegexUtils.isPhoneInvalid(phone)) {
        throw new BusinessException(ResultCode.PHONE_STYLE_ERROR);
    }

    // 2. 校验验证码
    Object cacheCode = session.getAttribute("code");  // 从 session 取出
    String code = loginForm.getCode();                // 前端提交的
    if (code == null || !code.equals(cacheCode)) {
        // 3. 不一致，报错
        throw new BusinessException(ResultCode.AUTH_CODE_ERROR);
    }

    // 4. 一致，根据手机号查询用户（MyBatis-Plus）
    User user = query().eq("phone", phone).one();

    // 5. 判断用户是否存在
    if (user == null) {
        // 6. 不存在，创建新用户并保存（自动注册）
        user = createUserWithPhone(phone);
    }

    // 7. 保存用户信息到 session（存 UserDTO，不是 User！）
    UserDTO userDTO = new UserDTO();
    BeanUtil.copyProperties(user, userDTO);   // 同名属性拷贝
    session.setAttribute("user", userDTO);
}

private User createUserWithPhone(String phone) {
    User user = new User();
    user.setPhone(phone);
    user.setNickName(USER_NICK_NAME_PREFIX + RandomUtil.randomNumbers(10));
    save(user);          // MyBatis-Plus 保存
    return user;         // 返回给上一步使用
}
```

> ⚠️ **细节**：判断顺序必须是 `code == null || !code.equals(cacheCode)`，**不能**写成 `!code.equals(cacheCode) || code == null`。`||` 从左往右短路求值，如果先执行 `code.equals(...)`，而 `code` 是 null，直接 **NPE**。

---

### 3.3 校验登录状态

**流程**：

```
开始 → 请求并携带 Cookie → 从 Session 获取用户 → 判断用户是否存在？
                                    不存在 → 拦截 → 结束
                                    存在   → 保存用户到 ThreadLocal → 放行 → 结束
```

**为什么不能每个 Controller 都写一遍**：业务越写越多，Controller 越来越多，每个都手动取 session、判空、拦截，重复且容易漏。

→ 用 SpringMVC 的**拦截器**，在**所有 Controller 之前**统一拦截：请求不再直接进 Controller，必须先过拦截器。把登录校验 + 用户信息放进 `ThreadLocal`，后续任何地方直接取（苍穹外卖也用过，见苍穹外卖 Day01 第 7 节）。

---

### 3.4 HttpSession 到底是什么

- `HttpSession` 代表"**当前这个用户（浏览器）的一次会话**"。
- 服务端为每个 session 分配唯一 ID，浏览器带着 Cookie（`JSESSIONID`）来，服务端据此认出"还是同一个人"。
- `session.setAttribute("code", code)`：往这个会话的**属性袋**里塞键值对，本质就是 `Map.put("code", code)`。
- **存在服务端内存里**（不是数据库、不是 Cookie），所以同一个 session 的后续请求都能取到：

```java
String savedCode = (String) session.getAttribute("code");   // 取出来比对
```

---

## 四、SpringMVC 拦截器基础

### 4.1 为什么要实现 HandlerInterceptor

拦截器是 Spring 提供的一种"**约定**"：你想在请求处理的某个环节插入代码，就实现这个接口，Spring 会在合适的时机自动调用你的方法。

`HandlerInterceptor` 就是一张"**插槽协议**"，上面留了三个位置：

```java
public interface HandlerInterceptor {
    boolean preHandle(...);    // 位置1：进控制器方法之前
    void postHandle(...);      // 位置2：控制器方法执行完、渲染视图之前
    void afterCompletion(...); // 位置3：整个请求彻底结束之后
}
```

写个类实现它，再注册到 `WebMvcConfigurer` 里，Spring 就会在每个请求的生命线上到这三个位置"敲门"。

> 这是 Spring 的**扩展点机制**：接口定义"在哪几个时机做事"，你的类负责"填什么内容"。
> 用法：新建类 → `implements HandlerInterceptor` → `Ctrl + I` 选 `preHandle` 和 `afterCompletion`。

---

### 4.2 一次请求的时间线

以一次请求 `/login` 为例：

```
浏览器请求
   ↓
[preHandle]          ← ① 还没进 Controller！登录校验在这
   ↓
Controller.login() 执行
   ↓
[postHandle]         ← ② 方法返回了，但响应还没写出去
   ↓
视图渲染（前后端分离项目直接输出 JSON）
   ↓
[afterCompletion]    ← ③ 所有事都做完了，收尾
```

---

### 4.3 preHandle（最关键）

| 项 | 说明 |
| --- | --- |
| 时机 | Controller 方法执行**之前** |
| 参数 | `request` / `response`（原始 HTTP 请求响应）、`handler`（请求最终交给谁处理，通常是 `HandlerMethod`，即"某个 Controller 的某个方法"） |
| 返回值 | **boolean 决定一切**：`true` 放行；`false` **中断请求链** |

**返回 false 的连锁反应**：

- 后面的 `postHandle`、Controller **统统不执行**
- 已经执行过的其他拦截器的 `preHandle` 仍会依次执行
- 已执行过的拦截器**不会**再调 `postHandle`（但会调 `afterCompletion`）

登录拦截就是靠"返回 false"实现：

```java
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
    User user = (User) request.getSession().getAttribute("user");
    if (user != null) {
        return true;   // 登录了，放行
    }
    // 没登录：直接吐 JSON，并中断请求
    response.setContentType("application/json;charset=utf-8");
    response.getWriter().write("{\"success\":false,\"errorMsg\":\"用户未登录\"}");
    return false;      // ← 这一行就是"拦截"本身
}
```

---

### 4.4 postHandle

- **时机**：Controller 方法**执行完之后**，响应渲染**之前**
- **多出的参数**：`model`（控制器往模型里塞的数据）、`mv`（`ModelAndView`）
- **用途**：往视图里统一补充数据（每个页面都要显示的"当前时间""用户昵称"）
- 前后端分离项目返回 JSON 不走视图渲染，**基本用不到**

---

### 4.5 afterCompletion

- **时机**：整个请求**彻底结束**（视图渲染完、响应写完）之后
- **多出的参数**：`ex` —— 请求过程中抛了异常会传进来，没异常就是 `null`
- **用途**：收尾统计 —— 记录请求耗时、**清理 ThreadLocal**、打日志

---

### 4.6 三者分工速记

| 方法 | 一句话 | 典型用途 |
| --- | --- | --- |
| `preHandle` | 进门检查 | **登录校验**（本项目就干这个）、权限、限流 |
| `postHandle` | 出门补妆 | 往响应里塞统一数据 |
| `afterCompletion` | 事后收尾 | 日志、耗时统计、清理资源 |

> 拦截器返回 false 后，被拦截的 Controller 方法不会执行 —— 这正是登录拦截的价值：**没登录的请求根本到不了 Controller 的方法体**。

---

### 4.7 拦截器 vs 过滤器

| 对比 | 过滤器 Filter | 拦截器 Interceptor |
| --- | --- | --- |
| 规范 | `javax.servlet.Filter`（Servlet 规范） | `HandlerInterceptor`（Spring MVC） |
| 生效层面 | Servlet 容器层，**连 Spring 都还没开始工作** | Spring MVC 内部 |
| 能拿到 | request / response | request / response + **handler（目标方法）+ 参数 + 模型** |
| 适用 | 编码、跨域、压缩等通用处理 | 登录校验、权限等**业务级**拦截 |

---

## 五、Session 版登录拦截器

### 5.1 LoginInterceptor

```java
package com.hmdp.utils;

import com.hmdp.dto.UserDTO;
import org.springframework.web.servlet.HandlerInterceptor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;

public class LoginInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        // 1. 获取 session
        HttpSession session = request.getSession();
        // 2. 获取 session 中的用户
        Object user = session.getAttribute("user");
        // 3. 判断用户是否存在
        if (user == null) {
            // 4. 不存在 → 拦截，返回 401 未授权
            response.setStatus(401);
            return false;
        }
        // 5. 存在 → 保存到 ThreadLocal（UserHolder 是自定义类）
        UserHolder.saveUser((UserDTO) user);
        // 6. 放行
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) throws Exception {
        // 移除用户，防止线程复用串号
        UserHolder.removeUser();
    }
}
```

**ThreadLocal 的作用**：`ThreadLocal` 为每个线程保存独立副本，Tomcat 一个请求由一个线程处理，所以"请求级"数据放这里最合适 —— 存一次，整条链路随处可取，线程之间互不干扰。

---

### 5.2 注册拦截器（MvcConfig）

在 `config` 包下新建配置类，实现 `WebMvcConfigurer`：

```java
@Configuration
public class MvcConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // registry 是注册器
        registry.addInterceptor(new LoginInterceptor())
                // 设置哪些路径不需要拦截
                .excludePathPatterns(
                        "/shop/**",       // 商铺详情，公共信息
                        "/shop-type/**",  // 商铺类型列表（美食/KTV），首页就展示
                        "/upload/**",     // 上传文件的公共接口
                        "/voucher/**",    // 优惠券的"查看"类接口（下单在 /voucher-order，未排除 ✅）
                        "/blog/hot",      // 热帖列表，公共浏览
                        "/user/code",     // 发验证码，登录前就要调，必须排除
                        "/user/login"     // 登录本身，必须排除
                );
    }
}
```

> 记忆：**"游客能看的一律排除，登录前后要调的接口必须排除"**。

---

### 5.3 为什么存 UserDTO 而不是 User

登录时先把 `user` 的属性拷到 `userDTO` 再存：

```java
UserDTO userDTO = new UserDTO();
BeanUtil.copyProperties(user, userDTO);   // 同名属性拷贝
session.setAttribute("user", userDTO);
```

**两个原因**：

1. **防敏感信息泄露** —— `User` 含密码、手机号等字段，一旦被序列化回前端就危险；`UserDTO` 只保留 id、昵称、头像等**非敏感字段**
2. **减少内存占用** —— 存的东西越少，序列化/反序列化越快、占空间越小

---

## 六、集群下的 Session 共享问题

**问题**：多台 Tomcat **不共享 session 存储空间**。请求被负载均衡切到另一台机器时，取不到 session，**数据丢失**（表现为"刚登录就掉线"）。

```
浏览器 → Nginx ─┬─→ Tomcat1（session 存在这）
                └─→ Tomcat2（没有 session！）→ 401
```

**替代方案必须满足 3 个条件**：

| 条件 | 原因 |
| --- | --- |
| 数据共享 | 多台服务访问同一份数据 |
| 内存结构 | 读写快，不能每次查库 |
| key-value 结构 | 按 key 精确定位，适合"按 token 找用户" |

**Redis 刚好全满足**。

---

## 七、方案二：基于 Redis 实现共享 Session 登录

### 7.1 为什么要换掉 Session

同 [[#六、集群下的 Session 共享问题]]：Session 绑在单台 Tomcat 内存上，集群下不可用。常见补救（session 复制、粘性会话）都不彻底且损耗性能，所以直接用 Redis 做**集中式会话存储**。

---

### 7.2 核心思路：把"两存一取"搬到 Redis

| Session 方案 | Redis 方案 |
| --- | --- |
| `session.setAttribute("code", code)` | `SET login:code:{phone} {code} EX 120` |
| `session.getAttribute("code")` | `GET login:code:{phone}` |
| `session.setAttribute("user", userDTO)` | `HSET login:token:{token} ...` + `EXPIRE 3600` |
| 浏览器自动带 `JSESSIONID` Cookie | 登录成功**返回 token**，前端每次请求放进请求头 **`authorization`** |
| 服务端拿 JSESSIONID 找 session | 服务端拿 **token 当 key** 去 Redis 找用户 |

> 一句话：**Session 的"会话 ID"换成了自己生成的 token，Session 的"存储"换成了 Redis。**
> 图中 `phone:xxx`、`token:xxx` 是简写，项目里统一加 `login:` 前缀区隔业务。

---

### 7.3 三条流程

#### 7.3.1 流程一：发送短信验证码

```
开始 → 提交手机号 → 校验手机号 ──不符合──┐
                        │符合           │
                        ↓               │(回退重填)
                    生成验证码 ←────────┘
                        ↓
              保存验证码到 Redis（以手机号为 key）
                        ↓
                    发送验证码
                        ↓
                       结束
```

Redis 中：`phone:13838411438 → 9527`

#### 7.3.2 流程二：短信验证码登录、注册

```
开始 → 提交手机号+验证码 → 校验验证码 ──不一致──┐
                              │一致             │
                              ↓                 │(回退)
                       根据手机号查询用户 ←──────┘
                              ↓
                        用户是否存在？
                    不存在 ↙        ↘ 存在
              创建新用户              │
                    ↓                 │
             保存用户到数据库          │
                    ↘                 ↙
        保存用户到 Redis（以随机 token 为 key，存用户数据）
                              ↓
                    返回 token 给客户端
                              ↓
                             结束
```

Redis 中：`token:fadfjklfweo → {name:lisi}`

**三个关键动作**：

1. 验证码**校验通过后立刻删除**（一次性，防重复使用）
2. 用户不存在则**自动注册**（手机号 + 随机昵称）
3. 登录成功**返回 token**，不返回用户敏感信息

#### 7.3.3 流程三：校验登录状态

```
开始 → 请求并携带 Token → 从 Redis 按 token 取用户 → 判断用户是否存在？
                                   存在 ↙              ↘ 不存在
                          保存用户到 ThreadLocal         拦截
                                    ↓                    ↓
                                  放行                  结束
                                    ↓
                                   结束
```

> 三张图其实是**同一个 Redis 在三个环节被读写**：验证码阶段用手机号当 key，登录态阶段用 token 当 key。

---

### 7.4 Redis Key 设计与 TTL

| 存什么 | 图中简写 | 项目规范写法 | value | 结构 | TTL |
| --- | --- | --- | --- | --- | --- |
| 短信验证码 | `phone:13838411438` | `login:code:13838411438` | `9527` | String | 2 分钟 |
| 登录用户 | `token:fadfjklfweo` | `login:token:fadfjklfweo` | `{name:lisi, ...}` | Hash（一个字段一个 field） | 1 小时 |

- 验证码 TTL 短：防止暴力破解
- 登录态 TTL 长：保证体验，且拦截器里可**续期**（滑动过期）

> **TTL 是固定的吗？活跃用户到期会被强制退出吗？**
>
> - TTL 是**常量写死的**（项目里 `LOGIN_USER_TTL`，当前设为 **3600 秒 = 1 小时**）
> - 默认是**固定过期**（absolute）：登录那一刻定死，之后用户**再活跃也不会自动续期**，到点 Redis 自动删 key → 下一个请求 401 → 相当于被"强制退出"，要重新登录（**正在处理的那个请求会跑完**，是**下一个**请求才开始 401）
> - 想要"活跃就一直保持登录"（滑动过期 / keep-alive）：在拦截器 `preHandle` 查到用户后**刷新一次 TTL**：
>
> ```java
> // 每活跃一次，TTL 重新计 3600s
> stringRedisTemplate.expire(RedisConstants.LOGIN_USER_KEY + token,
>         RedisConstants.LOGIN_USER_TTL, TimeUnit.SECONDS);
> ```
>
> 效果：1 小时内访问过 1 次就永不过期；1 小时不活跃才失效。代价几乎为零（`EXPIRE` 是 O(1)）。

---

### 7.5 关键实现点

#### ① token 怎么生成

**不能用手机号 / 用户 id 当 token**（可被伪造、可枚举），要用 UUID 去横线：

```java
String token = UUID.randomUUID().toString(true);
```

#### ② 验证码：写入与读取

```java
// 写：手机号当 key，2 分钟过期
stringRedisTemplate.opsForValue()
        .set(LOGIN_CODE_KEY + phone, code, LOGIN_CODE_TTL, TimeUnit.MINUTES);

// 读（替代 session.getAttribute("code")）
String cacheCode = stringRedisTemplate.opsForValue().get(LOGIN_CODE_KEY + phone);
```

#### ③ 用户信息为什么用 Hash 存

Hash 可以**只更新单个字段**（如改昵称），不用整体覆盖；代价是 **field 和 value 都必须是字符串**。

```java
stringRedisTemplate.opsForHash().putAll(LOGIN_USER_KEY + token, userMap);
stringRedisTemplate.expire(LOGIN_USER_KEY + token, LOGIN_USER_TTL, TimeUnit.SECONDS);
```

#### ④ ⭐ Long 类型怎么塞进 String 的 Hash？（高频卡点）

`StringRedisTemplate` 的 key 和 value 都是 **String 序列化器**，但 `UserDTO` 里 `id` 是 **Long**。直接 `putAll` 会抛类型转换异常，必须先把对象"洗"成 `Map<String, String>`。

```java
// 把 UserDTO 拆成 Map，并做两个"清洗"：丢掉 null 值、把所有值转成字符串
UserDTO userDTO = BeanUtil.copyProperties(user, UserDTO.class);
Map<String, Object> userMap = BeanUtil.beanToMap(userDTO, new HashMap<>(),
        CopyOptions.create()
                .setIgnoreNullValue(true)                              // ① 忽略 null
                .setFieldValueEditor((fieldName, fieldValue) -> fieldValue.toString())); // ② 全部转 String
```

三个 API 各自的作用：

| API | 作用 |
| --- | --- |
| `BeanUtil.copyProperties(user, UserDTO.class)` | `User` → `UserDTO`，同名属性拷贝，**只带非敏感字段** |
| `BeanUtil.beanToMap(dto, map, copyOptions)` | 对象 → `Map`，key 是字段名，value 是字段值 |
| `CopyOptions.setIgnoreNullValue(true)` | **忽略 null 值**：不往 Redis 写空字段，省内存，也避免 `toString()` 时 NPE |
| `FieldValueEditor` | 值编辑器：把 `Long` / `Integer` / `Date` 统一 `toString()`，满足 String 序列化器要求 |

**读回来怎么办？** 用 `fillBeanWithMap`，它会**按字段类型自动转换**（`"1"` → `Long 1`）：

```java
Map<Object, Object> entries = stringRedisTemplate.opsForHash().entries(LOGIN_USER_KEY + token);
if (entries.isEmpty()) {
    return null;   // 查不到用户
}
UserDTO dto = BeanUtil.fillBeanWithMap(entries, new UserDTO(), false);
```

> 结论：**存时手动 toString，取时自动转回去**。之所以要 toString，是因为 `StringRedisTemplate` 只认字符串；Hash 的 field 天生就是字符串，所以只能"降维"存。

#### ⑤ 拦截器怎么拿 token

Session 方案是 `request.getSession()`；Redis 方案改成**从请求头拿**：

```java
String token = request.getHeader("authorization");
```

前端登录成功后把 token 存 localStorage，之后每次请求都带上 `authorization` 头。

#### ⑥ ThreadLocal 必须清理

```java
@Override
public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                            Object handler, Exception ex) {
    UserHolder.removeUser();   // 一定要删
}
```

Tomcat 线程**池化复用**，不删会把上一个用户的信息带给下一个请求（**串号**），同时造成内存泄漏。

---

### 7.6 两个拦截器与执行顺序

| 拦截器 | order | 拦截范围 | 职责 |
| --- | --- | --- | --- |
| `RefreshTokenInterceptor` | 0 | **所有请求 `/**`** | 有 token 就查 Redis、**刷新 TTL**、把用户放进 ThreadLocal，然后**一律放行** |
| `LoginInterceptor` | 1 | 只拦需要登录的路径 | ThreadLocal 里没用户就返回 **401** + `return false` |

**为什么要拆两个**：刷新 TTL 要覆盖**所有**请求（包括游客能看的店铺列表），而登录校验只针对**部分**接口。合在一起会出现"逛首页也要求登录"。

**执行顺序与数据交接（order 值越小越先执行）**：

```
请求进来
   ↓
RefreshTokenInterceptor (order = 0, 拦 /**)     ← 先执行
   取 token → 查 Redis → 查到就存进 UserHolder(ThreadLocal)
           → 刷新 TTL(滑动续期) → 一律 return true 放行
   ↓
LoginInterceptor (order = 1, 拦除排除外的路径)   ← 后执行
   检查 ThreadLocal 有没有用户（UserHolder.getUser()）
   没有 → 401 拦截；有 → 放行
   ↓
Controller
```

**关键分工**：

| 拦截器 | 范围 | 查什么 | 拦不拦 |
| --- | --- | --- | --- |
| Refresh | 所有请求 `/**` | **查 Redis** | **不拦**，只"取用户 + 续命 + 存 ThreadLocal" |
| Login | 仅需登录的路径 | **查 ThreadLocal** | 没用户就 401 |

- **Refresh 必须先跑**：它把用户准备好放进 ThreadLocal，Login 才有得查 → 所以 `order` 必须 Refresh < Login
- 两个拦截器靠 **ThreadLocal 交接**：Refresh 存、Login 读，**Login 不直接查 Redis**
- `LoginInterceptor` 里的 `if (UserHolder.getUser() == null)` 读的就是 Refresh 刚放好的用户

---

### 7.7 Session 方案 vs Redis 方案

| 对比项 | Session | Redis |
| --- | --- | --- |
| 存储位置 | 单台 Tomcat 内存 | 独立中间件，所有服务共享 |
| 集群支持 | ❌ 需 session 复制 / 粘性会话 | ✅ 天然支持 |
| 服务重启 | 数据丢失 | 数据还在（RDB/AOF） |
| 过期控制 | 由容器配置 | 灵活，可续期（滑动过期） |
| 客户端标识 | 浏览器自动带 Cookie | 前端手动带 token 请求头 |
| 额外依赖 | 无 | 多一个 Redis |
| 适合场景 | 单机 / 小规模 | 集群 / 分布式 |

---

### 7.8 易错点清单

| ❌ 易错 | ✅ 正确 |
| --- | --- |
| 用手机号 / 用户 id 当 token | 用 `UUID.randomUUID().toString(true)`，防伪造 |
| 验证码校验完不删 | 校验成功后 `delete`，防重复使用 |
| token 不设过期时间 | 必须设 TTL，否则 Redis 无限膨胀 |
| 只写一个拦截器 | Refresh（全部路径 + 续期）+ Login（部分路径 + 校验） |
| `afterCompletion` 忘了 `removeUser()` | 必须清理，否则线程复用串号 + 内存泄漏 |
| 把 `User` 整个存进 Redis | 存 `UserDTO`，只留 id / nickName / icon 等非敏感字段 |
| `return false` 却不写响应 | 返回 401 或 JSON，否则前端拿到空响应 |
| Long 直接塞进 Hash | 先 `beanToMap` + `toString()`，读时 `fillBeanWithMap` 自动转回 |

---

### 7.9 动手实现清单（按顺序做）

1. `RedisConstants` 增加常量：`LOGIN_CODE_KEY`、`LOGIN_CODE_TTL`、`LOGIN_USER_KEY`、`LOGIN_USER_TTL`
2. `UserServiceImpl.sendCode`：`session.setAttribute` → `stringRedisTemplate.opsForValue().set(...)`
3. `UserServiceImpl.login`：验证码改从 Redis 读 → 校验后删除 → 生成 token → `UserDTO` 转 Map 存 Hash → 设 TTL → **返回 token**
4. `UserController` 的登录接口：返回值从 `Result.ok()` 改成 `Result.ok(token)`
5. 新建 `RefreshTokenInterceptor`（拦 `/**`，order 0）
6. 改造 `LoginInterceptor`：不再查 session，只查 ThreadLocal（order 1）
7. `MvcConfig` 注册两个拦截器并设置 `order`

---

## 八、面试考点

### Q1：为什么不用 Session，改用 Redis？（必问）

**答**：Session 存在单台 Tomcat 内存里，**集群下多台机器不共享**，请求被负载均衡切走就丢失登录态。补救方案（session 复制、粘性会话）要么浪费内存要么不彻底。Redis 是独立中间件，满足**数据共享 + 内存存储 + KV 结构**三个条件，天然适合做集中式会话存储，还支持过期时间和集群。

### Q2：登录流程完整说一遍

**答**：
1. 提交手机号 → 校验格式 → 生成 6 位验证码 → 存 Redis（key = `login:code:手机号`，TTL 2 分钟）→ 发送
2. 提交手机号 + 验证码 → 从 Redis 取验证码比对 → 一致则删掉 → 按手机号查用户 → 不存在自动注册 → 生成 token → 用 Hash 存用户信息（key = `login:token:xxx`，TTL 1 小时）→ **返回 token**
3. 后续请求带 `authorization` 头 → 拦截器按 token 查 Redis → 查到就存 ThreadLocal 并续期 → 放行

### Q3：token 怎么生成？为什么不用用户 id？

**答**：用 `UUID.randomUUID().toString(true)`。用用户 id 或手机号当 token，**可枚举、可伪造**，攻击者改个数字就能冒充别人；UUID 不可预测。

### Q4：拦截器里为什么用 ThreadLocal？有什么风险？

**答**：ThreadLocal 给每个线程一份独立副本，一个请求由一个线程处理，所以能在"请求级"共享用户信息，不用层层传参，线程间互不干扰。
**风险**：Tomcat 线程池复用线程，**不清理会把上一个用户的数据带给下一个请求（串号），还会内存泄漏** → 必须在 `afterCompletion` 里 `remove()`。

### Q5：拦截器的 `preHandle` 返回 false 会发生什么？

**答**：中断请求链 —— 目标 Controller **不会执行**，`postHandle` 也不会执行；已执行过 `preHandle` 的拦截器仍会依次执行 `afterCompletion`。这正是"未登录请求到不了 Controller"的实现方式。

### Q6：为什么要用两个拦截器？order 有什么讲究？

**答**：职责和范围不同。`RefreshTokenInterceptor`（order 0，拦 `/**`）负责**取用户 + 刷新 TTL + 存 ThreadLocal，从不拦截**；`LoginInterceptor`（order 1，只拦需登录路径）负责**检查 ThreadLocal 决定是否 401**。合在一起会导致"游客浏览店铺列表也被要求登录"，而只留 Refresh 又没人做校验。order 必须 Refresh < Login，否则 Login 检查时 ThreadLocal 还是空的。

### Q7：拦截器和过滤器有什么区别？

**答**：Filter 属于 Servlet 规范，在容器层生效，**Spring 还没初始化**，拿不到 Controller 和参数；Interceptor 属于 Spring MVC，能拿到 `handler`、模型数据，适合做登录/权限这类业务拦截。

### Q8：为什么存 UserDTO 而不是 User？

**答**：`User` 含密码、手机号等敏感字段，存进 session/Redis 有泄露风险；`UserDTO` 只保留 id、昵称、头像等展示字段，同时减小存储体积、降低序列化开销。

### Q9：为什么 Redis 里用 Hash 存用户，而不是 String 存 JSON？

**答**：Hash 支持**单字段更新**（改昵称只 `HSET` 一个 field），String + JSON 必须整体读出→改→写回，既费带宽又有并发覆盖风险。**代价**：Hash 的 value 只能是字符串，Long 等类型要 `toString()`，读取时再靠 `fillBeanWithMap` 转回来。

### Q10：用户一直活跃，token 会过期吗？

**答**：默认 TTL 是**固定过期**（登录时定死 3600s），不续期就会到期强制退出。想要"活跃即续命"，在 `RefreshTokenInterceptor` 查到用户后执行一次 `EXPIRE` 重置 TTL，实现**滑动过期**，`EXPIRE` 是 O(1)，代价可忽略。

---

## 九、今日重点

### 9.1 必记 5 件事

1. **登录的本质 = 服务端记住你是谁**：Session 靠 `JSESSIONID`，Redis 靠 **token**。
2. **集群 → Session 失效 → 换 Redis**，这是整个改造的动机，面试必答。
3. **三个 Redis key**：`login:code:{phone}`（2min）、`login:token:{token}`（1h），一个短一个长。
4. **两个拦截器 + ThreadLocal**：Refresh（`/**`，取用户 + 续期，不拦）→ Login（部分路径，查 ThreadLocal，401）。
5. **ThreadLocal 必须 remove**，否则线程复用串号 + 内存泄漏。

### 9.2 关键 API 速记

```java
// 验证码
stringRedisTemplate.opsForValue().set(LOGIN_CODE_KEY + phone, code, LOGIN_CODE_TTL, TimeUnit.MINUTES);
stringRedisTemplate.opsForValue().get(LOGIN_CODE_KEY + phone);

// token
String token = UUID.randomUUID().toString(true);

// 用户（Hash）
Map<String, Object> userMap = BeanUtil.beanToMap(userDTO, new HashMap<>(),
        CopyOptions.create().setIgnoreNullValue(true)
                .setFieldValueEditor((k, v) -> v.toString()));
stringRedisTemplate.opsForHash().putAll(LOGIN_USER_KEY + token, userMap);

// 取用户
Map<Object, Object> entries = stringRedisTemplate.opsForHash().entries(LOGIN_USER_KEY + token);
UserDTO dto = BeanUtil.fillBeanWithMap(entries, new UserDTO(), false);

// 续期
stringRedisTemplate.expire(LOGIN_USER_KEY + token, LOGIN_USER_TTL, TimeUnit.SECONDS);
```

### 9.3 易错点速查

| ❌ | ✅ |
| --- | --- |
| `!code.equals(cacheCode) \|\| code == null` | `code == null \|\| !code.equals(cacheCode)`（短路求值防 NPE） |
| 用手机号当 token | UUID |
| 单拦截器搞定所有事 | Refresh + Login 两个，order 0 / 1 |
| 存 `User` 到会话 | 存 `UserDTO` |
| Long 直接进 Hash | `toString()` 存，`fillBeanWithMap` 取 |

### 9.4 明日预告

- 商户查询缓存：缓存更新策略、缓存穿透 / 击穿 / 雪崩
- Redis 在业务里的三件套：**缓存 + 分布式锁 + 计数器**
