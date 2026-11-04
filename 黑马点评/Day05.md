# Day05 · 缓存雪崩与缓存击穿

> **本日路线**：缓存雪崩 → 缓存击穿（互斥锁 / 逻辑过期）→ 两种方案的代码落地 → 对比选型
>
> **一句话**：缓存穿透是"数据根本不存在"，雪崩是"大量 key 同时失效"，击穿是"一个热点 key 突然失效"。前两者靠"多存/错开"，击穿靠"**抢锁重建**（互斥锁）或"**先给旧数据、后台重建**（逻辑过期）。

---

## 目录

- [[#一、缓存雪崩]]
- [[#二、缓存击穿]]
- [[#三、互斥锁解决缓存击穿（Shop）]]
- [[#四、逻辑过期解决缓存击穿（ShopType）]]
- [[#五、两种方案对比]]
- [[#六、面试考点]]
- [[#七、今日重点]]

---

## 一、缓存雪崩

### 1.1 是什么

**同一时间段大量缓存 key 同时失效**，或者 **Redis 服务宕机**，导致大量请求直接打到数据库，带来巨大压力。

```
大量 key 同时过期 / Redis 挂了
        ↓
所有请求都绕过缓存
        ↓
数据库瞬间被打满
```

和缓存穿透的区别：穿透是"**单个不存在的数据**每次都要查库"，雪崩是"**大批 key 同时没了**"，冲击面更大。

### 1.2 解决方案

| # | 方案 | 说明 |
| --- | --- | --- |
| 1 | **TTL 加随机值** | 给不同 key 的过期时间加随机值，避免同一时刻集体失效 |
| 2 | **Redis 集群** | 提高服务可用性，避免单点宕机 |
| 3 | **降级限流** | 给缓存业务加降级、限流策略，保护数据库 |
| 4 | **多级缓存** | 在 Redis 之外再加本地缓存等，分摊压力 |

> 口诀：**"错开时间、多备几台、兜底限流、多加一层。"**

---

## 二、缓存击穿

### 2.1 是什么

缓存击穿也叫**热点 key 问题**：一个被**高并发访问**、并且**缓存重建业务较复杂**的 key 突然失效了，无数请求会在瞬间涌向数据库。

理解：某一时刻，一个被高并发访问的 key 忽然失效，大量线程同时去数据库查询并重建缓存，而重建业务又比较复杂，于是数据库瞬间承受巨大冲击。

**关键特征（和穿透区分）**：

| | 缓存穿透 | 缓存击穿 |
| --- | --- | --- |
| 数据是否存在 | **不存在** | **存在** |
| 缓存状态 | 一直没有 | 有，但**刚好过期** |
| 特点 | 每次请求都打库 | **瞬间**大量请求同时打库重建 |

### 2.2 方案一：互斥锁

**原理**：查询未命中时，先获取互斥锁；**获取成功（其他线程获取不到）才能查数据库、重建缓存**，写回 Redis 后再释放锁。期间其他线程获取锁失败，就休眠一会儿，再从头来（查缓存 → 抢锁）。

```
线程1：未命中 → 抢锁成功 → 查库 → 重建缓存 → 放锁
线程2：未命中 → 抢锁失败 → 休眠 → 重试（查缓存 → 抢锁）
```

**优点**：

1. 没有额外内存消耗
2. 保证一致性
3. 实现简单

**缺点**：

1. 线程需要等待，性能受影响
2. 可能有死锁风险（靠锁的过期时间缓解）

### 2.3 方案二：逻辑过期

先给缓存数据设置一个**逻辑过期时间**：一个由应用程序自己维护、**存储在缓存数据内部**的"过期时间戳"。它和 Redis 自带的物理过期时间（TTL）是**两套完全独立的机制**——key 本身不设 TTL、永不过期，是否过期由数据里的时间戳判断。

**流程**：线程 A 查询缓存，发现逻辑时间已过期 → 获取互斥锁。获取成功后，**开启一个新线程 B** 去查数据库、重建缓存、重置逻辑过期时间，完成后释放锁。在此期间，线程 A 直接返回**过期数据**；其他线程发现逻辑时间过期、抢锁失败，也返回过期数据。直到线程 B 完成，后续请求才能命中最新且未过期的缓存。

```
线程1：发现过期 → 抢锁成功 → 开线程B（异步重建）→ 立即返回旧数据
线程2：发现过期 → 抢锁失败 → 直接返回旧数据
线程B：查库 → 重建缓存 → 重置过期时间 → 放锁
```

**优点**：线程无需等待，性能较好
**缺点**：

1. 不保证一致性（返回的是旧数据）
2. 有额外内存消耗（要存过期时间）
3. 实现复杂

---

## 三、互斥锁解决缓存击穿（Shop）

对应代码：`ShopServiceImpl#queryWithMutex(Long id)`

### 3.1 整体流程

1. 从 Redis 查商铺缓存
2. 命中真实数据（`isNotBlank`）→ 反序列化直接返回
3. 命中空值标记（`shopJson != null`，即 `""`）→ 返回 null（店铺不存在）
4. 两者都不是 → 开始缓存重建
   - 4.1 尝试获取互斥锁（`SETNX`）
   - 4.2 获取失败 → `Thread.sleep(50)` 后递归重试
   - 4.3 获取成功 → **DoubleCheck 再查一次 Redis**：有就返回，没有再查库
   - 4.4 查库：不存在 → 写空值 `""`（短 TTL）；存在 → 写缓存（30 分钟）
5. `finally` 释放锁
6. 返回

### 3.2 核心代码

```java
private Shop queryWithMutex(Long id){
    // 1. 查缓存
    String shopJson = stringRedisTemplate.opsForValue().get(RedisConstants.CACHE_SHOP_KEY + id);
    // 2. 命中真实数据 → 直接返回
    if(StrUtil.isNotBlank(shopJson)){
        return JSONUtil.toBean(shopJson, Shop.class);
    }
    // 3. 命中空值标记 "" → 不存在
    if (shopJson != null) {
        return null;
    }

    // 4. 缓存重建
    String lockKey = RedisConstants.LOCK_SHOP_KEY + id;
    Shop shop = null;
    try {
        // 4.1 获取互斥锁
        boolean isLock = tryLock(lockKey);
        // 4.2 获取失败 → 休眠重试
        if(!isLock){
            Thread.sleep(50);
            return queryWithMutex(id);
        }
        // 4.3 获取成功 → DoubleCheck：再查一次 Redis 拿最新值
        String shopJson1 = stringRedisTemplate.opsForValue().get(RedisConstants.CACHE_SHOP_KEY + id);
        if (StrUtil.isNotBlank(shopJson1)){
            return JSONUtil.toBean(shopJson1, Shop.class);  // 别人已重建好，直接用
        }
        if(shopJson1 != null){
            return null;                                     // 别人写了空值标记
        }
        // 4.4 仍然没有 → 查数据库
        shop = getById(id);
        Thread.sleep(200);                                    // 模拟重建耗时
        if (shop == null) {
            // 数据库也没有 → 写空值标记，防缓存穿透
            stringRedisTemplate.opsForValue().set(
                RedisConstants.CACHE_SHOP_KEY + id, "",
                RedisConstants.CACHE_NULL_TTL, TimeUnit.MINUTES);
            return null;
        }
        // 4.5 存在 → 写缓存
        stringRedisTemplate.opsForValue().set(
            RedisConstants.CACHE_SHOP_KEY + id, JSONUtil.toJsonStr(shop),
            RedisConstants.CACHE_SHOP_TTL, TimeUnit.MINUTES);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    } finally {
        // 5. 释放锁
        unlock(lockKey);
    }
    return shop;
}
```

### 3.3 关键点

| # | 点 | 说明 |
| --- | --- | --- |
| 1 | **DoubleCheck 的位置** | 必须在 `tryLock` 成功之后、`getById` 之前。从"发现没缓存"到"抢到锁"之间可能等了好几轮，前一个持锁线程可能已经重建好了，抢到锁后再查一次，有就直接用 |
| 2 | **空值标记用 `""`** | Redis 没有 null，`StringRedisTemplate` 的 value 也不能为 null；写 `""` + 短 TTL（`CACHE_NULL_TTL` = 2 分钟），再用 `isNotBlank` + `!= null` 区分"真未命中"和"空值标记" |
| 3 | **`tryLock` 用 `setIfAbsent`** | 等价于 `SET key value NX EX 10`，同时完成"key 不存在才设置（互斥）"和"设置过期时间（防死锁）" |
| 4 | **`BooleanUtil.isTrue(flag)`** | `setIfAbsent` 返回包装类型 `Boolean`，直接 `return flag` 会自动拆箱，可能 NPE |

### 3.4 踩坑：`finally` 释放锁

当前写法把 `return queryWithMutex(id)` 放在 `try` 内，**没抢到锁的线程返回时 `finally` 照样执行 `unlock`**，会把别的线程正持有的锁删掉，互斥失效。

```
线程2 抢锁失败 → 休眠 → return（递归重试）
                    ↓ finally 仍然执行
                 unlock(lockKey)  ← 把线程1 的锁删了！
```

**正解**：让 `try/finally` 只包住「成功拿到锁之后」的代码，或用一个 `isLock` 标志控制是否释放。

---

## 四、逻辑过期解决缓存击穿（ShopType）

对应代码：`ShopTypeServiceImpl`

### 4.1 前提：缓存预热

逻辑过期要求缓存**一开始就存在**（key 永不过期），所以第一次要把数据手动写进 Redis，并带上逻辑过期时间。通常放在测试类里调用：

```java
private void saveShopType2Redis(Long expireSeconds) {
    // 1. 查询分类数据
    List<ShopType> list = query().orderByAsc("sort").list();

    // 2. 封装逻辑过期时间（data 字段是 Object，List 直接塞）
    RedisData redisData = new RedisData();
    redisData.setData(list);
    redisData.setExpireTime(LocalDateTime.now().plusSeconds(expireSeconds));

    // 3. 写入 Redis（注意：不设物理 TTL）
    stringRedisTemplate.opsForValue().set(
        RedisConstants.CACHE_SHOPTYPE_KEY, JSONUtil.toJsonStr(redisData));
}
```

写入后的 JSON 结构：

```json
{ "expireTime": "2026-10-22T21:30:00", "data": [ {...}, {...} ] }
```

### 4.2 读取流程

1. 从 Redis 查缓存
2. 缓存为空 → 返回空列表
3. 命中 → 把 JSON 还原成 `RedisData`，取出 `expireTime` 和 `data`（`data` 实际是 `JSONArray`）
4. 判断 `expireTime`：
   - **未过期** → 直接返回数据
   - **已过期** → 尝试获取互斥锁
5. 抢到锁 → **DoubleCheck** 再查一次，判断是否已被别人重建
6. 仍未重建 → 提交到线程池，**异步**重建缓存，**线程内释放锁**
7. 主线程不管重建成功与否，直接返回**旧数据**

### 4.3 核心代码

```java
// 线程池：用于异步重建缓存
private static final ExecutorService CACHE_REBUILD_EXECUTOR = Executors.newFixedThreadPool(10);

private List<ShopType> queryWithLogicalExpire(){
    // 1. 查缓存
    String cacheShopType = stringRedisTemplate.opsForValue().get(RedisConstants.CACHE_SHOPTYPE_KEY);
    // 2. key 不存在 → 返回空列表
    if (StrUtil.isBlank(cacheShopType)) {
        return Collections.emptyList();
    }

    // 3. JSON → RedisData → List<ShopType>
    RedisData redisData = JSONUtil.toBean(cacheShopType, RedisData.class);
    List<ShopType> list = JSONUtil.toList((JSONArray) redisData.getData(), ShopType.class);
    LocalDateTime expireTime = redisData.getExpireTime();

    // 4. 未过期 → 直接返回
    if (expireTime.isAfter(LocalDateTime.now())) {
        return list;
    }

    // 5. 已过期 → 抢互斥锁
    String lockKey = RedisConstants.LOCK_SHOPTYPE_KEY;
    boolean isLock = tryLock(lockKey);
    if (isLock) {
        // 5.1 DoubleCheck：锁内重查，判断是否已被别人重建（看 expireTime）
        String latestJson = stringRedisTemplate.opsForValue().get(RedisConstants.CACHE_SHOPTYPE_KEY);
        if (StrUtil.isNotBlank(latestJson)) {
            RedisData latestData = JSONUtil.toBean(latestJson, RedisData.class);
            if (latestData.getExpireTime().isAfter(LocalDateTime.now())) {
                // 已不过期：别人建好了 → 释放锁，返回新数据
                unlock(lockKey);
                return JSONUtil.toList((JSONArray) latestData.getData(), ShopType.class);
            }
        }
        // 5.2 仍过期 → 开启独立线程重建缓存
        CACHE_REBUILD_EXECUTOR.submit(() -> {
            try {
                saveShopType2Redis(20L);
            } catch (Exception e) {
                throw new RuntimeException(e);
            } finally {
                unlock(lockKey);   // 锁必须在线程内释放
            }
        });
    }

    // 6. 无论是否重建，先返回旧数据
    return list;
}
```

### 4.4 关键点

| # | 点 | 说明 |
| --- | --- | --- |
| 1 | **`data` 是 `Object`** | `RedisData.data` 声明为 `Object`，`setData(list)` 没问题；序列化后 `data` 是数组 |
| 2 | **读取两步转换** | `toBean(json, RedisData.class)` 拆外层壳；`getData()` 是 **`JSONArray`**，再用 `toList((JSONArray) data, ShopType.class)` 转实体列表 |
| 3 | **DoubleCheck 看"过期"不看"有没有值"** | 逻辑过期的 key 永远有值（无物理 TTL），`isBlank` 永远 false；必须解析 `expireTime` 再比较 |
| 4 | **锁必须在线程内释放** | 主线程立刻 `return list`，若在主线程 `unlock`，重建还没完锁就放了，互斥失效 |
| 5 | **主线程返回旧数据** | 逻辑过期的本质：**用一致性换性能**，用户不等待 |

### 4.5 易错点

| ❌ 易错 | ✅ 正解 |
| --- | --- |
| 把逻辑过期逻辑写进 `queryWithPassThrough()` | 方法名和内容要对上，逻辑过期放 `queryWithLogicalExpire()` |
| `return list;` 之后还留着查库/写缓存代码 | 删掉，否则是**不可达代码（Unreachable statement）**，编译报错 |
| `(JSONObject) redisData.getData()` 强转列表 | 列表是 `JSONArray`，强转 `JSONObject` 会 `ClassCastException` |
| DoubleCheck 用 `isBlank` 判断 | 逻辑过期的 key 永远有值，要判断 `expireTime.isAfter(now)` |
| 锁在主线程释放 | 必须在线程内 `finally` 释放 |
| `LOCK_SHOPTYPE_KEY = "lock:shop:"` | 会和商铺锁冲突，应为 `"lock:shop-type:"` |

---

## 五、两种方案对比

| 对比项 | 互斥锁 | 逻辑过期 |
| --- | --- | --- |
| 核心思路 | 抢到锁才能重建，其他线程**等待** | 返回旧数据，**异步**重建 |
| 一致性 | **强一致** | 最终一致（短时间返回旧数据） |
| 性能 | 线程等待，较差 | **无需等待，较好** |
| 内存 | 无额外开销 | 需存 `expireTime`，有额外开销 |
| 实现复杂度 | **简单** | 复杂 |
| 死锁风险 | 有（靠 TTL 缓解） | 有（靠 TTL 缓解） |
| 适用场景 | 对一致性要求高、能接受等待 | 对性能要求高、能接受短暂旧数据 |

> 选型口诀：**"要准就抢锁，要快就逻辑过期。"**

---

## 六、面试考点

### Q1：缓存雪崩是什么？怎么解决？

**答**：同一时间段**大量 key 同时失效**或 **Redis 宕机**，请求全部打到数据库。解决：① TTL 加随机值（错开失效时间）；② Redis 集群（提高可用性）；③ 降级限流（保护数据库）；④ 多级缓存（分摊压力）。

### Q2：缓存击穿是什么？和缓存穿透有什么区别？

**答**：击穿是**热点 key 突然失效**，大量请求同时查库重建；穿透是**数据根本不存在**，每次请求都查库。关键区别：击穿的数据**存在**、只是缓存过期；穿透的数据**不存在**。

### Q3：互斥锁和逻辑过期怎么选？

**答**：

- **互斥锁**：强一致、实现简单，但线程要等待，性能差。适合**对一致性要求高**的场景。
- **逻辑过期**：返回旧数据、异步重建，性能好，但不保证一致、实现复杂。适合**对性能要求高、能接受短暂旧数据**的场景。

### Q4：逻辑过期的 key 为什么不设 TTL？

**答**：逻辑过期的核心就是**key 永不过期**（物理上），是否过期由数据里的 `expireTime` 逻辑判断。如果设了 TTL，key 到期被 Redis 删掉，就变成"缓存不存在"，逻辑过期机制就失效了。

### Q5：逻辑过期的 DoubleCheck 为什么判断 `expireTime` 而不是 `isBlank`？

**答**：逻辑过期的 key 永远有值（没有物理 TTL），`isBlank` 永远返回 false，判断不出"是否过期"。正确做法是重新解析出 `RedisData`，比较 `expireTime` 和当前时间：未过期说明别人已重建，直接返回；仍过期才重建。

### Q6：为什么锁必须在线程内释放？

**答**：逻辑过期里主线程拿到旧数据后**立刻返回**。如果 `unlock` 写在主线程，锁会在后台重建完成前就被释放，其他线程又能抢到锁重复重建，互斥失效。所以 `unlock` 要放在**重建线程的 finally** 里。

### Q7：互斥锁的 `finally` 有什么坑？

**答**：如果获取锁失败的线程也执行 `finally` 的 `unlock`，会把**别的线程正持有的锁**删掉，互斥失效。解决：`try/finally` 只包住"拿到锁之后"的代码，或用 `isLock` 标志控制是否释放。

### Q8：`RedisData` 里的 List 怎么反序列化？

**答**：`getData()` 返回 `Object`，实际类型是 `JSONArray`（存 List 时）或 `JSONObject`（存单个对象时）。

- 单个对象：`JSONUtil.toBean((JSONObject) data, Shop.class)`
- 列表：`JSONUtil.toList((JSONArray) data, ShopType.class)`

---

## 七、今日重点

### 7.1 必记 5 件事

1. **三种问题分清**：穿透（数据不存在）、击穿（热点 key 失效）、雪崩（大量 key 同时失效）。
2. **击穿两招**：互斥锁（等待重建、强一致）、逻辑过期（返回旧数据、异步重建、高性能）。
3. **DoubleCheck 必做**：抢到锁后先再查一次缓存，避免重复重建。
4. **锁的释放要看位置**：互斥锁注意 `finally` 误删别人的锁；逻辑过期的锁必须在**线程内**释放。
5. **逻辑过期的数据是 `RedisData`**：读取要先 `toBean` 拆壳，再把 `JSONArray` 用 `toList` 转成实体列表。

### 7.2 代码 / API 速记

```java
// 互斥锁：setIfAbsent = SET NX EX
Boolean flag = stringRedisTemplate.opsForValue()
        .setIfAbsent(lockKey, "1", 10, TimeUnit.SECONDS);
return BooleanUtil.isTrue(flag);

// 逻辑过期：写入
RedisData redisData = new RedisData();
redisData.setData(list);
redisData.setExpireTime(LocalDateTime.now().plusSeconds(20));
stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonStr(redisData));

// 逻辑过期：读取
RedisData data = JSONUtil.toBean(json, RedisData.class);
List<ShopType> list = JSONUtil.toList((JSONArray) data.getData(), ShopType.class);

// 判断逻辑过期
if (data.getExpireTime().isAfter(LocalDateTime.now())) {
    // 未过期
}
```

### 7.3 易错点清单

| ❌ 易错 | ✅ 正确 |
| --- | --- |
| 三种问题混淆 | 穿透=不存在，击穿=热点失效，雪崩=大量失效 |
| 逻辑过期写进穿透方法 | 方法名和逻辑要对上 |
| `return` 后残留代码 | 删掉，否则不可达代码编译报错 |
| `getData()` 强转 `JSONObject` | 列表是 `JSONArray`，用 `toList` |
| DoubleCheck 判断"有没有值" | 逻辑过期要判断"过没过期" |
| 锁在主线程释放 | 在线程内 `finally` 释放 |
| 逻辑过期的 key 设了 TTL | **不设 TTL**，靠 `expireTime` 判断 |

### 7.4 还没写到的部分（待补）

| 知识点 | 状态 |
| --- | --- |
| 缓存工具封装 | 未整理（`CacheClient` + 函数式参数，把四种方案封装成通用工具） |
| 分布式锁 | 未整理（`SETNX` 手写锁的缺陷 → Redisson） |
