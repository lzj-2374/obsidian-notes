# Day06 · Redis 缓存工具类 CacheClient

> **本日路线**：Spring 依赖注入 → CacheClient 的四个方法 → 泛型 + Function 回调 → 逻辑过期的通用化 → 锁与 DoubleCheck
>
> **一句话**：把缓存穿透 / 逻辑过期的重复代码抽成通用工具类，靠「泛型 + Function 回调」让工具类不认识业务，靠「锁 + DoubleCheck + 异步重建」保证击穿场景的正确性。

---

## 目录

- [[#一、构造器注入（依赖注入）]]
- [[#二、CacheClient 的四个方法]]
- [[#三、set / setWithLogicalExpire]]
- [[#四、queryWithPassThrough（缓存穿透）]]
- [[#五、queryWithLogicalExpire（缓存击穿）]]
- [[#六、通用化的三个细节]]
- [[#七、面试考点]]
- [[#八、今日重点]]

---

## 一、构造器注入（依赖注入）

`CacheClient` 的写法：

```java
@Component
public class CacheClient {

    private final StringRedisTemplate stringRedisTemplate;

    public CacheClient(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }
}
```

**为什么不用 `@Autowired` / `@Resource`：**

| # | 原因 | 说明 |
| --- | --- | --- |
| 1 | 可以 `final` | 依赖创建后不可修改，对象状态稳定；字段注入不能 final |
| 2 | 依赖显式 | 看构造器就知道这个类需要什么 |
| 3 | 不会出现半成品 | 对象创建时依赖必须传入；手动 new 也不会得到 null 字段 |
| 4 | 方便单元测试 | `new CacheClient(mockTemplate)` 即可，不用启动 Spring |
| 5 | 循环依赖早暴露 | A↔B 循环依赖在启动时就报错 |

**`@Autowired` vs `@Resource`：**

- `@Autowired`：Spring 提供，默认**按类型**注入
- `@Resource`：JSR-250 标准注解，默认**按名称**（字段名）注入，找不到再按类型

**两个注意点：**

- 只有一个构造器时，`@Autowired` 可以省略（Spring 4.3+ 自动识别）
- 构造器注入 ≠ 注册 Bean，类上仍然需要 `@Component`

---

## 二、CacheClient 的四个方法

|  | 普通 TTL | 逻辑过期 |
| --- | --- | --- |
| **写** | `set` | `setWithLogicalExpire` |
| **读** | `queryWithPassThrough` | `queryWithLogicalExpire` |

**调用关系：**

```
queryWithPassThrough   ──写缓存──>  set
queryWithLogicalExpire ──重建──>  setWithLogicalExpire
```

| 方法 | 职责 |
| --- | --- |
| `set` | 普通缓存写入（JSON + 物理 TTL） |
| `setWithLogicalExpire` | 逻辑过期缓存写入（`RedisData` 包装，不设 TTL） |
| `queryWithPassThrough` | 解决**缓存穿透**的查询 |
| `queryWithLogicalExpire` | 解决**缓存击穿**的查询（逻辑过期） |

`set*` 负责"怎么存"，`query*` 负责"怎么查 + 什么时候回填/重建"。

---

## 三、set / setWithLogicalExpire

```java
public void set(String key, Object value, Long time, TimeUnit unit){
    stringRedisTemplate.opsForValue()
            .set(key, JSONUtil.toJsonStr(value), time, unit);
}

public void setWithLogicalExpire(String key, Object value, Long time, TimeUnit unit){
    RedisData redisData = new RedisData();
    redisData.setData(value);
    redisData.setExpireTime(LocalDateTime.now().plusSeconds(unit.toSeconds(time)));

    stringRedisTemplate.opsForValue()
            .set(key, JSONUtil.toJsonStr(redisData));   // 不设物理 TTL
}
```

**关键：**

- `set`：普通缓存，带物理 TTL
- `setWithLogicalExpire`：**不设物理 TTL**，过期靠 `expireTime`
- 最后写入的必须是 `redisData`，不是 `value`（写 value 就丢了过期时间）

---

## 四、queryWithPassThrough（缓存穿透）

### 4.1 签名

```java
public <R, ID> R queryWithPassThrough(
        String keyPrefix, ID id, Class<R> type,
        Function<ID, R> dbFallBack, Long time, TimeUnit unit)
```

### 4.2 流程

1. `key = keyPrefix + id`
2. 从 Redis 查缓存
3. `isNotBlank` → 命中真实数据，`toBean` 返回
4. `!= null`（空值标记 `""`）→ 返回 null
5. `dbFallBack.apply(id)` 查数据库
6. 查不到 → 写 `""` + 短 TTL（`CACHE_NULL_TTL`），返回 null
7. 查到 → 调用 `set` 写缓存，返回

### 4.3 关键点

- **`<R, ID>`**：`R` 返回类型，`ID` 主键类型
- **`Class<R> type`**：反序列化时告诉 Hutool 转成什么
- **`Function<ID, R>`**：把「怎么查库」作为参数传进来
- **`apply(id)`**：执行调用方传进来的方法（如 `this::getById`）
- **两次判断**：`isNotBlank` 判真实数据，`!= null` 判空值标记

### 4.4 Function 是什么

`Function` 是 JDK 自带的函数式接口，只有一个抽象方法：

```java
R apply(T t);   // 给我一个 T，还我一个 R
```

`Function<Long, Shop>` 读作「输入 Long，输出 Shop 的函数」，可以用 lambda 或方法引用实现：

```java
Function<Long, Shop> f1 = id -> shopService.getById(id);
Function<Long, Shop> f2 = shopService::getById;
```

在 CacheClient 里，它的作用是把「查库逻辑」**作为参数传进来**，工具类只负责"什么时候调用"，具体查什么由业务决定。

---

## 五、queryWithLogicalExpire（缓存击穿）

### 5.1 流程

1. `key = keyPrefix + id`，查缓存
2. `isBlank` → 直接返回 null（逻辑过期假设缓存已预热）
3. `toBean` 成 `RedisData`，取出 `data` 和 `expireTime`
4. `expireTime` 未过期 → 直接返回
5. 已过期 → `tryLock("lock:" + keyPrefix + id)`
6. 抢到锁 → **DoubleCheck**：重查缓存，若已不过期 → `unlock` 并返回新数据
7. 仍过期 → 提交线程池异步重建，`finally` 里 `unlock`
8. 主线程返回**旧数据**

### 5.2 关键点

- **DoubleCheck 判 `expireTime`，不判 `isBlank`**（逻辑过期的 key 永远有值）
- 锁必须在**线程内**释放（主线程会立刻返回）
- 主线程返回旧数据：用一致性换性能
- 重建调用 `setWithLogicalExpire`，写入新的过期时间

### 5.3 锁的两个工具方法

```java
private boolean tryLock(String key){
    Boolean flag = stringRedisTemplate.opsForValue()
            .setIfAbsent(key, "1", 10, TimeUnit.SECONDS);   // SET NX EX
    return BooleanUtil.isTrue(flag);                        // 防拆箱 NPE
}

private void unlock(String key){
    stringRedisTemplate.delete(key);
}
```

---

## 六、通用化的三个细节

### 6.1 锁 key 要独立命名空间

```java
String lockKey = "lock:" + keyPrefix + id;
```

不能直接用 `keyPrefix + id` —— 那和缓存 key 完全一样：

- `tryLock` 会把缓存内容覆盖成 `"1"`
- `unlock` 会把缓存数据一起删掉

### 6.2 List 适配：`(JSONObject)` 强转不成立

单个对象时 `getData()` 是 `JSONObject`，列表时是 `JSONArray`。通用写法：

```java
String json = JSONUtil.toJsonStr(data);
if (List.class.isAssignableFrom(type)) {
    return (R) JSONUtil.parseArray(json);   // JSONArray 实现了 List
}
return JSONUtil.toBean(json, type);
```

### 6.3 泛型擦除拿不到 List 元素类型

方法签名只有 `Class<R> type`（`List.class`），没有元素类型信息，所以列表只能返回 `JSONArray`（元素是 `JSONObject`）。如果需要真正的 `List<ShopType>`，要额外传元素 `Class`，或单独写列表版本的查询方法。

### 6.4 调用侧的两个例子

```java
// Shop：按 id 查单个对象
Shop shop = cacheClient.queryWithPassThrough(
        RedisConstants.CACHE_SHOP_KEY, id, Shop.class,
        this::getById, CACHE_SHOP_TTL, TimeUnit.MINUTES);

// ShopType：查列表（没有 id，用空串占位）
List<ShopType> list = cacheClient.queryWithPassThrough(
        RedisConstants.CACHE_SHOPTYPE_KEY, "", List.class,
        (id) -> query().orderByAsc("sort").list(),
        CACHE_SHOPTYPE_TTL, TimeUnit.MINUTES);
```

---

## 七、面试考点

### Q1：为什么推荐构造器注入，而不是字段注入？

**答**：构造器注入可以 `final`（不可变）、依赖显式、对象创建时依赖必须齐全（不会 NPE）、方便单元测试、循环依赖能在启动时暴露。字段注入在这几方面都不如它。

### Q2：@Autowired 和 @Resource 有什么区别？

**答**：

- `@Autowired` 是 Spring 的注解，**默认按类型**注入；多个同类型 Bean 时可配合 `@Qualifier` 指定名称。
- `@Resource` 是 JSR-250 标准注解，**默认按名称**注入（字段名），找不到同名 Bean 再按类型。

### Q3：缓存工具类为什么要用泛型 + Function 回调？

**答**：工具类要能被所有业务复用，不能写死 `getById`。

- `<R, ID>` 让返回类型和主键类型参数化；
- `Function<ID, R>` 让「查库逻辑」由调用方传入；
- CacheClient 只负责"什么时候查/写缓存"，不关心具体查什么表。

### Q4：缓存穿透的代码为什么要判断两次？

**答**：

- `isNotBlank`：判断有没有真实数据；
- `!= null`：判断是不是命中了空值标记 `""`。

顺序不能反：先拦真实数据，再区分"缓存了空值"和"根本没缓存"。空值标记必须用 `""`（Redis 没有 null，`StringRedisTemplate` 的 value 也不能为 null）。

### Q5：逻辑过期的 DoubleCheck 为什么判 `expireTime`，不判 `isBlank`？

**答**：逻辑过期的 key 永远有值（没有物理 TTL），`isBlank` 永远返回 false，判断不出"是否过期"。正确做法是重新解析出 `RedisData`，比较 `expireTime` 和当前时间。

### Q6：锁为什么不能用缓存 key？为什么必须在线程内释放？

**答**：

- 锁 key 和缓存 key 相同会互相覆盖/删除：`tryLock` 把缓存写成 `"1"`，`unlock` 把缓存删掉。所以要加 `"lock:"` 独立命名空间。
- 逻辑过期里主线程拿到旧数据后立刻返回，`unlock` 写在主线程会在后台重建完成前就释放锁，其他线程又能抢锁重复重建。所以要在重建线程的 `finally` 里释放。

### Q7：逻辑过期的 key 为什么不设 TTL？

**答**：逻辑过期的核心是 key 永不过期（物理上），是否过期由数据里的 `expireTime` 判断。设了 TTL，key 到期被 Redis 删除，就变成"缓存不存在"，逻辑过期机制失效。

### Q8：`RedisData` 里的 List 怎么反序列化？

**答**：`getData()` 返回 `Object`，实际类型：

- 单个对象 → `JSONObject` → `JSONUtil.toBean((JSONObject) data, Shop.class)`
- 列表 → `JSONArray` → `JSONUtil.toList((JSONArray) data, ShopType.class)`

通用工具里只有 `Class<R> type`，拿不到 List 元素类型，所以只能返回 `JSONArray`（元素是 `JSONObject`）。

---

## 八、今日重点

### 8.1 必记 5 件事

1. **四个方法的分工**：`set` / `setWithLogicalExpire` / `queryWithPassThrough` / `queryWithLogicalExpire`。
2. **泛型 + Function 回调**：工具类不认识业务，查库逻辑由调用方传入。
3. **穿透的两次判断**：`isNotBlank` 判真实数据，`!= null` 判空值标记。
4. **逻辑过期的 DoubleCheck**：抢到锁后重查，判 `expireTime` 不判 `isBlank`。
5. **锁的三条铁律**：`setIfAbsent`（NX+EX）、独立 key、持锁线程内释放。

### 8.2 代码 / API 速记

```java
// 普通缓存
stringRedisTemplate.opsForValue()
        .set(keyPrefix + id, JSONUtil.toJsonStr(value), ttl, TimeUnit.MINUTES);

// 逻辑过期缓存（不设 TTL）
RedisData redisData = new RedisData();
redisData.setData(value);
redisData.setExpireTime(LocalDateTime.now().plusSeconds(unit.toSeconds(time)));
stringRedisTemplate.opsForValue()
        .set(key, JSONUtil.toJsonStr(redisData));

// 互斥锁
Boolean flag = stringRedisTemplate.opsForValue()
        .setIfAbsent("lock:" + keyPrefix + id, "1", 10, TimeUnit.SECONDS);

// 列表反序列化
List<ShopType> list = JSONUtil.toList((JSONArray) redisData.getData(), ShopType.class);
```

### 8.3 易错点清单

| ❌ 易错 | ✅ 正确 |
| --- | --- |
| 锁 key 直接用 `keyPrefix + id` | 加 `"lock:"` 前缀，独立命名空间 |
| `setWithLogicalExpire` 里写 `value` | 写 `redisData` |
| DoubleCheck 用 `isBlank` 判断 | 用 `expireTime.isAfter(now)` |
| 主线程里 `unlock` | 重建线程的 `finally` 里 `unlock` |
| `(JSONObject) getData()` 处理 List | 转 JSON 字符串后按 `type` 分支 |
| 逻辑过期的 key 设物理 TTL | 不设 TTL，靠 `expireTime` |
| 以为构造器注入就不用 `@Component` | Bean 注册和依赖注入是两件事 |
| 在非 Spring 环境手动 new 带字段注入的类 | 字段为 null，会 NPE；构造器注入才能直接 new |

### 8.4 待补

| 知识点 | 状态 |
| --- | --- |
| 优惠券秒杀 | 待补（全局唯一 ID、库存超卖、一人一单） |
| 分布式锁 | 待补（`SETNX` 手写锁的缺陷 → Redisson） |
| 达人探店 / 关注 Feed 流 | 待补 |
