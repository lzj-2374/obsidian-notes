# Day02 · Jedis 与 SpringDataRedis

> **一句话概括**：Java 操作 Redis 有两套常用客户端 —— 原生 **Jedis**（简单、线程不安全、必须配连接池）和 Spring 官方的 **SpringDataRedis**（**RedisTemplate** 统一 API，SpringBoot 默认用 **Lettuce**）。今天就一件事：**把 Java 和 Redis 连起来，并且把序列化这个坑填平。**

---

## 目录

- [[#一、Jedis 客户端]]
- [[#二、SpringDataRedis]]
- [[#三、面试考点汇总]]
- [[#四、今日重点]]

---

## 一、Jedis 客户端

### 1.1 使用步骤

#### 步骤 1：引入依赖

```xml
<!-- jedis依赖 -->
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>4.3.0</version>
</dependency>

<!-- 连接池必须配套：commons-pool2，JedisPool 连接池需要 -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

#### 步骤 2：建立连接

```java
private Jedis jedis;

@BeforeEach
void setUp() {
    // 建立连接
    jedis = new Jedis("192.168.150.101", 6379);
    // 设置密码
    jedis.auth("123321");
    // 选择库（默认 0~15 共 16 个库）
    jedis.select(0);
}
```

#### 步骤 3：测试与释放资源

```java
// 测试 String 类型
@Test
void testString() {
    // 插入数据，方法名对应 redis 命令（set / get / hset / lpush ...）
    String result = jedis.set("name", "张三");
    System.out.println("result = " + result);   // OK
    // 获取数据
    String name = jedis.get("name");
    System.out.println("name = " + name);       // 张三
}

// 释放资源
@AfterEach
void tearDown() {
    if (jedis != null) {
        jedis.close();
    }
}
```

> **规律**：Jedis 的**方法名 = Redis 命令名**，会用 redis-cli 就会用 Jedis（`jedis.set`、`jedis.hgetAll`、`jedis.zadd`…）。

---

### 1.2 为什么一定要用连接池？

| 问题 | 说明 |
| --- | --- |
| **Jedis 线程不安全** | 一个 Jedis 实例不能多线程共享，否则命令会串包 |
| **频繁创建/销毁连接开销大** | 每次 `new Jedis()` 都要走一遍 TCP 三次握手 + 认证 |

**结论**：连接池复用连接、控制并发上限、避免连接泄漏。

---

### 1.3 JedisPool 连接池（单例工厂）

```java
public class JedisConnectionFactory {
    private static final JedisPool jedisPool;

    static {
        JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
        // 最大连接
        jedisPoolConfig.setMaxTotal(8);
        // 最大空闲连接
        jedisPoolConfig.setMaxIdle(8);
        // 最小空闲连接
        jedisPoolConfig.setMinIdle(0);
        // 设置最长等待时间，ms
        jedisPoolConfig.setMaxWaitMillis(200);

        jedisPool = new JedisPool(jedisPoolConfig, "192.168.150.101", 6379,
                1000, "123321");
    }

    // 获取 Jedis 对象
    public static Jedis getJedis() {
        return jedisPool.getResource();
    }
}
```

#### 参数逐个拆解

| 代码 | 含义 |
| --- | --- |
| `setMaxTotal(8)` | 连接池**最大总连接数**，最多同时 8 个连接 |
| `setMaxIdle(8)` | **最大空闲连接**，空闲时最多保留 8 个 |
| `setMinIdle(0)` | **最小空闲连接**，空闲时最低维持 0 个 |
| `setMaxWaitMillis(200)` | 借连接最多等 200ms，**超时报错**（不是一直等） |

```java
new JedisPool(配置, ip, 端口, 超时时间, 密码)
//                          ↑ 1000ms 是连接超时，不是读取超时
```

| 方法 | 作用 |
| --- | --- |
| `jedisPool.getResource()` | **从连接池中获取一个 Jedis 连接** |
| `static {}` 代码块 | 类加载时**只执行一次**，一次性初始化连接池（单例思想） |

#### 使用示例

```java
@Test
void testPool() {
    // 从连接池获取连接
    Jedis jedis = JedisConnectionFactory.getJedis();
    jedis.set("name", "李四");
    System.out.println(jedis.get("name"));
    // 使用完毕归还连接
    jedis.close();
}
```

> ⭐ **重点**：连接池模式下 `jedis.close()` **不是销毁 TCP 连接**，而是**把连接归还回连接池**供别人复用。所以「用完一定要 close」，不是「用完就断开」。

---

## 二、SpringDataRedis

### 2.1 定位

> **SpringData** 是 Spring 的数据操作模块，集成各类数据库；**SpringDataRedis** 就是 SpringData 针对 Redis 的集成模块。
> 官网：<https://spring.io/projects/spring-data-redis>

### 2.2 核心特性

1. 整合多种 Redis 客户端：**Lettuce、Jedis**
2. 提供 **RedisTemplate**，统一 API 操作 Redis
3. 支持 Redis 发布订阅模型
4. 支持 Redis 哨兵、Redis 集群
5. 支持基于 Lettuce 的响应式编程
6. 支持多种序列化 / 反序列化：JDK、JSON、字符串、Spring 对象
7. 支持基于 Redis 的 JDK Collection 实现

---

### 2.3 快速入门

#### 步骤 1：引入依赖

```xml
<!-- Redis 依赖 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<!-- 连接池依赖 -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

#### 步骤 2：配置文件（application.yml）

```yaml
spring:
  redis:
    host: 192.168.150.101
    port: 6379
    password: your-password
    lettuce:
      pool:
        max-active: 8   # 最大连接
        max-idle: 8     # 最大空闲连接
        min-idle: 0     # 最小空闲连接
        max-wait: 100   # 连接等待时间（ms，建议写成 100ms 更明确）
```

> ⚠️ **版本差异坑**：Spring Boot 2.x 前缀是 `spring.redis.*`；**Spring Boot 3.x 起改成 `spring.data.redis.*`**。写错前缀不报错，只会静默用默认值 `localhost:6379`，然后连不上。

#### ⭐ 步骤 2.5：网络前置（不做 100% 连不上）

1. **改 `redis.conf`，允许外部 IP 访问**：把 `bind 127.0.0.1` 改成 `bind 0.0.0.0`
2. **关闭 Ubuntu 防火墙，或放行 6379 端口**

```bash
# 关闭防火墙（Ubuntu）
sudo ufw disable
```

3. **重启 redis 服务**（改配置不重启不生效）

```bash
sudo systemctl restart redis
```

> 还有两个常见原因：**保护模式**（`protected-mode no`）和**密码错误**。用 `redis-cli -h <ip> -p 6379 -a <pwd> ping` 在宿主机先验证一遍，能通再写 Java 代码。

#### 步骤 3：注入 RedisTemplate

```java
@Autowired
private RedisTemplate redisTemplate;
```

---

### 2.4 RedisTemplate 的 API 分组

`RedisTemplate` 把 5 种数据类型的操作**拆分成不同操作对象**，用 `opsForXxx()` 拿到对应 API：

| API | 返回值类型 | 作用 |
| --- | --- | --- |
| `redisTemplate.opsForValue()` | `ValueOperations` | 操作 String 字符串类型 |
| `redisTemplate.opsForHash()` | `HashOperations` | 操作 Hash 哈希类型 |
| `redisTemplate.opsForList()` | `ListOperations` | 操作 List 列表类型 |
| `redisTemplate.opsForSet()` | `SetOperations` | 操作 Set 集合类型 |
| `redisTemplate.opsForZSet()` | `ZSetOperations` | 操作 SortedSet（有序集合 / ZSet） |
| `redisTemplate` 本身 | — | 执行通用 Redis 命令（`delete`、`expire`、`hasKey`…） |

```java
// 操作 String
ValueOperations<String, String> ops = redisTemplate.opsForValue();
ops.set("name", "张三");
String name = ops.get("name");
```

> **记忆法**：`Value` 对应 String（String 类型在 RedisTemplate 里叫 **Value**）；`opsForZSet` 才是有序集合，别和 `opsForSet` 混。

---

### 2.5 序列化问题（今天的核心坑）

#### 问题现象

`RedisTemplate` 可以接受任意 `Object` 作为值，写入前会先**序列化**。默认采用 **JDK 序列化**，在 `redis-cli` 里看到的是：

```
\xAC\xED\x00\x05t\x00\x04name
```

**两个缺点**：

1. **可读性差**：人看不懂，客户端工具里也是一堆二进制
2. **内存占用较大**：JDK 序列化会带类描述信息

#### 解决方案 A：自定义 RedisTemplate 改 JSON 序列化

```java
@Bean
public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory)
        throws UnknownHostException {
    // 创建 Template
    RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
    // 设置连接工厂
    redisTemplate.setConnectionFactory(redisConnectionFactory);
    // 设置序列化工具
    GenericJackson2JsonRedisSerializer jsonRedisSerializer =
            new GenericJackson2JsonRedisSerializer();
    // key 和 hashKey 采用 String 序列化
    redisTemplate.setKeySerializer(RedisSerializer.string());
    redisTemplate.setHashKeySerializer(RedisSerializer.string());
    // value 和 hashValue 采用 JSON 序列化
    redisTemplate.setValueSerializer(jsonRedisSerializer);
    redisTemplate.setHashValueSerializer(jsonRedisSerializer);
    return redisTemplate;
}
```

**代价**：为了反序列化时知道对象类型，JSON 序列化器会把**类的 class 类型写进 JSON**，额外占内存：

```json
{"@class":"com.heima.pojo.User","name":"张三","age":18}
```

#### 解决方案 B：StringRedisTemplate（省内存的推荐做法）

为了省内存，实际项目**不一定要用 JSON 序列化器**：可以统一用 **String 序列化器**，只存 String 类型的 key 和 value；需要存 Java 对象时**手动序列化 / 反序列化**。

Spring 直接提供了 `StringRedisTemplate`，key 和 value 默认都是 String 序列化方式，**省去自己写配置类**：

```java
@Autowired
private StringRedisTemplate stringRedisTemplate;

// 存对象：手动 JSON 序列化
User user = new User("张三", 18);
stringRedisTemplate.opsForValue().set("user:1", JSON.toJSONString(user));

// 取对象：手动 JSON 反序列化
String json = stringRedisTemplate.opsForValue().get("user:1");
User u = JSON.parseObject(json, User.class);
```

> 这就是**黑马点评项目里的标准做法** —— 项目里注入的就是 `StringRedisTemplate`，配合 `JSONUtil`（hutool）手动转换。

---

### 2.6 StringRedisTemplate vs 自定义 RedisTemplate

#### 三种序列化器在 Redis 里的样子

| 序列化器 | Redis 里存的样子 | 特点 |
| --- | --- | --- |
| **JDK 序列化**（RedisTemplate 默认） | `\xAC\xED\x00\x05...` 二进制乱码 | 不可读、跨语言不通用 |
| **String 序列化**（StringRedisTemplate） | 纯文本 `杰哥` | 可读，只能存字符串 |
| **JSON 序列化**（GenericJackson2JsonRedisSerializer） | `{"@class":"java.lang.String","value":"杰哥"}` | 可读，能存对象 |

> "乱码"其实分两种：JDK 二进制是**真乱码**；JSON 带引号花括号看着怪但**正常**，读回来能还原。

#### 两个核心 Bean

```
StringRedisTemplate          ← Spring Boot 自带，key/value 都是 String 序列化器
RedisTemplate<String,Object> ← 自己写配置类，value 用 JSON 序列化器
```

#### 怎么选

| 要存什么 | 用哪个 |
| --- | --- |
| 纯字符串（含"手动序列化的 JSON 字符串"） | `StringRedisTemplate`（自带，不用配） |
| Java 对象（想自动序列化，图省事） | 自定义 `RedisTemplate<String, Object>`（JSON 序列化） |

> 自己定义了同名 `redisTemplate` Bean 后，会**覆盖 Spring Boot 自动配置的那个** —— 启动日志 Condition Report 里 `RedisAutoConfiguration#redisTemplate` 显示 `Did not match` 就是证据。

---

### 2.7 踩坑记录

#### 坑 1：缺 `jackson-databind` 依赖

Spring Boot 高版本（4.x）的 `starter-data-redis` 不再自动带 Jackson：

```
ClassNotFoundException: com/fasterxml/jackson/databind/jsontype/TypeResolverBuilder
```

解决：手动加依赖（**版本由 parent 管理，不用写 version**）：

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
</dependency>
```

#### 坑 2：`application.yml` 里写了 properties 语法

`application.yml` 必须用 YAML 缩进，**不能写** `spring.application.name=xxx`，IDEA 会标红。properties 语法只属于 `application.properties`。

#### 坑 3：读写序列化器不一致

同一个 key 写用 JSON、读用 String（或反过来），数据直接废掉。

> 规则：**同一批 key 必须用同一个模板读写**；换了序列化方式，先把旧 key `DEL` 掉。

```bash
# 换序列化方案后，清掉旧数据
redis-cli keys "heima:*" | xargs redis-cli del
```

---

## 三、面试考点汇总

### 3.1 核心结论（背这四条）

1. SpringBoot 整合 Redis **默认使用 Lettuce**，不是 Jedis。
2. `RedisTemplate` 是 SpringDataRedis 提供的核心操作对象，封装所有 Redis 命令。
3. 默认序列化是 **JDK 序列化**，缺点是**可读性差 + 占用空间大**，项目一般改成 **JSON 序列化** 或直接用 `StringRedisTemplate`。
4. **Lettuce 基于 Netty，线程安全**；**Jedis 线程不安全，必须配合连接池**。

### 3.2 Jedis vs Lettuce（高频对比题）

| 维度 | Jedis | Lettuce |
| --- | --- | --- |
| 底层 | 阻塞 IO（Socket 直连） | 基于 **Netty**，事件驱动 |
| 线程安全 | ❌ 不安全，一连接一线程 | ✅ 安全，多线程可共享一个连接 |
| 连接数 | 并发高时连接数暴涨 | 单连接多路复用，连接数少 |
| 是否需连接池 | **必须** | 可选（一般还是配，用于限流） |
| SpringBoot | 手动引入 | **默认** |

> 答题模板：Jedis 是"一个线程一个连接"的模型，简单直接但并发高时连接开销大；Lettuce 基于 Netty 做异步多路复用，线程安全、连接数少，所以 SpringBoot 默认选它。

### 3.3 API 对应关系（送分题）

| Redis 类型 | 方法 |
| --- | --- |
| String | `opsForValue()` |
| Hash | `opsForHash()` |
| List | `opsForList()` |
| Set | `opsForSet()` |
| ZSet | `opsForZSet()` |

> 陷阱：String 类型在 RedisTemplate 里叫 **Value**，不叫 `opsForString`。

### 3.4 追问：`jedis.close()` 到底做了什么？

**答**：不是关闭 TCP 连接。从连接池 `getResource()` 拿到的 Jedis 是**池内连接的包装**，`close()` 会**把连接归还给连接池**（内部调用 `returnResource`），供后续请求复用；只有**直连模式**（`new Jedis(host, port)`）下 `close()` 才是真正断开 TCP。

---

## 四、今日重点

### 4.1 必记 5 件事

1. **Jedis 方法名 = Redis 命令名**，`commons-pool2` 是连接池的必备依赖。
2. **Jedis 线程不安全**，必须用 `JedisPool`；`close()` 是**归还连接**不是断开。
3. **SpringBoot 默认客户端是 Lettuce**（基于 Netty、线程安全），不是 Jedis。
4. **RedisTemplate 默认 JDK 序列化 → 乱码 + 占内存**，解决两条路：自定义 JSON 序列化器 / 直接用 `StringRedisTemplate` + 手动 JSON。
5. **连不上先查三件事**：`bind 0.0.0.0`、防火墙放行 6379、`restart redis`（外加密码和 `protected-mode`）。

### 4.2 代码模板速记

```java
// ---------- Jedis 直连 ----------
Jedis jedis = new Jedis("192.168.150.101", 6379);
jedis.auth("123321");
jedis.select(0);
jedis.close();

// ---------- Jedis 连接池 ----------
Jedis jedis = JedisConnectionFactory.getJedis();
jedis.close();   // 归还

// ---------- SpringDataRedis ----------
@Autowired
private StringRedisTemplate stringRedisTemplate;

stringRedisTemplate.opsForValue().set("name", "张三");
String name = stringRedisTemplate.opsForValue().get("name");
```

### 4.3 易错点清单

| ❌ 易错 | ✅ 正确 |
| --- | --- |
| 认为 `jedis.close()` 是关闭连接 | 池模式下是**归还连接** |
| 用 `@Autowried` | `@Autowired`（注解拼写，IDEA 会提示） |
| 认为 RedisTemplate 默认 Lettuce 序列化 | 默认是 **JDK 序列化**，客户端才是 Lettuce |
| `opsForString()` | 没有这个 API，是 `opsForValue()` |
| yml 配完不重启 Redis | 改了 `redis.conf` 必须 `systemctl restart redis` |
| 同一个 key 换序列化器继续读写 | 先 `DEL` 旧 key，且同一批 key 用同一个模板 |
| 只加 `starter-data-redis` 就用连接池 | 还要 `commons-pool2` |

### 4.4 明日预告

- 基于 Redis 的**实战应用**：缓存（商户查询缓存）、缓存穿透/击穿/雪崩
- 黑马点评里 `StringRedisTemplate` 的真实用法：`opsForValue` 存 JSON、`opsForHash` 存对象字段、`opsForZSet` 做排行榜、`opsForSet` 做点赞去重
