# Day04 笔记

> [!info] 项目
> 苍穹外卖 · 第 4 天
>
> 涉及内容：Spring Data Redis、RedisTemplate 配置、Result 泛型使用、HttpClient
>
> 上一篇：[[苍穹外卖/day03]]

- **tags**: #苍穹外卖 #SpringBoot #Redis

---

## 1. 在 Java 中操作 Redis（Spring Data Redis）

### 1.1 导入 Maven 依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

### 1.2 配置数据源

Redis 默认创建了 16 个数据库（db0-db15），不同数据库之间数据完全隔离：

```yaml
spring:
  redis:
    host: localhost
    port: 6379
    password: your-password
    database: 10
```

### 1.3 编写配置类，创建 RedisTemplate 对象

```java
@Configuration
@Slf4j
public class RedisConfiguration {

    @Bean
    public RedisTemplate redisTemplate(RedisConnectionFactory redisConnectionFactory){
        log.info("开始创建redis模板类...");
        RedisTemplate redisTemplate = new RedisTemplate();
        // 设置Key的序列化器，默认为JdkSerializationRedisSerializer
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        return redisTemplate;
    }
}
```

### 1.4 通过 RedisTemplate 对象操作 Redis

（此处按需求补充具体操作代码）

---

## 2. Result 泛型使用

**问题**：为啥在 Controller 层有的方法 `public Result xxx() {}` 中的 Result 要加泛型，有的不用加？

### 2.1 Result\<T\> 泛型的含义

从 `Result.java` 可以看到，`Result<T>` 是一个泛型类，T 代表 data 字段的数据类型。

### 2.2 什么时候加泛型？

需要返回数据时，加泛型说明 data 的类型：

```java
// 返回分页数据
public Result<PageResult> pageQuery(...) {
    return Result.success(pageResult);  // data = pageResult
}

// 返回单个对象
public Result<SetmealVO> getById(...) {
    return Result.success(setmealVO);   // data = setmealVO
}
```

### 2.3 什么时候不加泛型？

不需要返回数据时（只返回操作成功/失败），使用原始类型 Result：

```java
public Result save(...) {
    setmealService.save(setmealDTO);
    return Result.success();  // 没有 data，只返回 code=1
}

public Result delete(...) {
    setmealService.deleteBatch(ids);
    return Result.success();  // 同上
}
```

---

## 3. HttpClient

**HttpClient**：可以用来提供高效的，最新的功能丰富的支持HTTP协议的客户端编程工具包，并且支持HTTP协议最新的版本和建议。
```
<dependency>
	<groupId>org. apache. httpcomponents</groupId>
	<artifactId>httpclient</artifactId>
	<version>4.5.13</version>
</dependency>
```

核心API:

. HttpClient
. HttpClients
. CloseableHttpClient
. HttpGet
. HttpPost

发送请求的步骤：
1，创建HttpClient对象
2，创建Http请求对象（put，get。。。）
3，调用HttpClient的execute方法发送请求

---

## 4. 面试问题

> [!question] 🎤 面试题
> **问题**：Spring Boot 操作 Redis 时，为什么通常要把 RedisTemplate 的 Key 序列化器改成 StringRedisSerializer？默认的有问题吗？Result<T> 的泛型什么时候加、什么时候不加？

**参考答案**：

1. **RedisTemplate 序列化器**：
   - 默认使用 `JdkSerializationRedisSerializer`，会把 key 序列化成**带乱码的二进制**（如 `\xac\xed\x00\x05t...`），在 `redis-cli` 里既看不懂也不便排查、甚至无法与其他语言客户端互通；
   - 改成 `StringRedisSerializer` 后，key 在 Redis 里直观可读。生产上 key 和 value 一般会分别设置（value 仍可用 JSON 序列化器）。
2. **Result<T> 泛型**：
   - **需要返回数据时加**泛型，指明 `data` 的类型，如 `Result<PageResult>`、`Result<SetmealVO>`；
   - **只表示操作成功/失败、不返回数据时不加**，直接 `Result.success()`（无 data，只返回 code=1）。