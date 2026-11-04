# Day09 · 基于 Redis 的分布式锁（Lua 原子释放）

> **本日路线**：Redis 分布式锁实现思路 → SimpleRedisLock 获取/释放锁 → 发现锁误删问题 → 三代写法演进 → Lua 脚本原子释放
>
> **一句话**：`SET NX EX` 抢锁 + value 存「UUID + 线程 id」+ 释放前用 Lua 把「判断是不是自己的锁」和「删除」合成一个不可打断的操作——补上 Java 三步之间那段封不住的空隙。

---

## 目录

- [[#一、基于 Redis 的分布式锁实现思路]]
- [[#二、为什么需要 Lua：三代写法演进]]
- [[#三、SimpleRedisLock 完整实现]]
- [[#四、unlock.lua 逐行解析]]
- [[#五、面试考点]]
- [[#六、今日重点]]

---

## 一、基于 Redis 的分布式锁实现思路

Redis 提供了 Lua 脚本功能：**在一个脚本中编写多条 Redis 命令，确保多条命令执行的原子性。**

**实现思路**：

1. 利用 `SET NX EX` 获取锁，设置过期时间，**value 保存线程标识**
2. 释放锁先判断线程标识是否与自己一致，一致则删除锁

**特性**：

1. 利用 `SET NX` 满足**互斥性**
2. 利用 `SET EX` 保证故障时锁依然能释放，**避免死锁**，提高安全性
3. 利用 Redis 集群保证**高可用和高并发**

**释放锁的业务流程**：

1. 获取锁中的线程标识
2. 判断是否与指定标识（当前线程标识）一致
3. 一致则释放锁（删除）
4. 不一致则什么都不做

`RedisTemplate` 调用 Lua 脚本的 API：

```java
public <T> T execute(RedisScript<T> script, List<K> keys, Object... args) {
    return scriptExecutor.execute(script, keys, args);
}
```

---

## 二、为什么需要 Lua：三代写法演进

**一句话：解决「释放锁时误删别人的锁」。**

「判断锁是不是自己的」和「删除锁」是两步，Java 端执行时中间可能被插入其他操作；Lua 让这两步在 Redis 里一次执行完、不被打断。

| 版本 | 写法 | 问题 |
| --- | --- | --- |
| ① 直接删 | `delete(key)` | 连是不是自己的锁都不判断 → 业务超时锁自动释放后，必删别人的锁 |
| ② Java 判断 | `get` → `equals` → `delete` | 判断通过了，但删除前锁恰好到期被别人抢走 → 删的还是别人的锁 |
| ③ Lua 脚本 | `get` + 比较 + `del` 在 Redis 内一次执行 | 脚本执行期间不会插入其他命令 → 空档消失，安全 |

所以 Lua 不是为了"写起来高级"，而是为了补上 ② 里那段**无法用 Java 代码封住的空隙**。

**② 的时间线（面试必背）**：

```
线程A：get 到 value 是自己的 → equals 通过
       ↓（此时锁的 TTL 到期，Redis 自动删除）
线程B：setIfAbsent 成功，拿到锁
线程A：delete 执行 → 删掉的是线程B的锁
```

Redis 执行 Lua 是**单线程串行**的，脚本运行期间不会插入其他命令，所以「比较 + 删除」不会被切开。

**「原子性」的准确含义：不是把多步变成一步，而是让多步在执行期间不被打断。**

---

## 三、SimpleRedisLock 完整实现

### 3.1 锁标识：为什么线程 id 前面要加 UUID

```java
private static final String KEY_PREFIX = "lock:";
private static final String ID_PREFIX = UUID.randomUUID().toString(true) + "-";
```

`toString(true)` 是 Hutool 的写法，`true` 表示去掉横线。

线程 id **只在单个 JVM 内唯一**，集群两台机器都有「线程 37」。只存 `37` 的话，A 机器的线程 37 可能把自己当成 B 机器线程 37 的锁主人，误删别人的锁。

加上 UUID 后 value 变成 `4f2a...c1-37`，跨机器不会撞。通用套路：**本地唯一 + 进程唯一 = 全局唯一**。

### 3.2 static 块：脚本只加载一次

```java
private static final DefaultRedisScript<Long> UNLOCK_SCRIPT;
static {
    UNLOCK_SCRIPT = new DefaultRedisScript<>();
    UNLOCK_SCRIPT.setLocation(new ClassPathResource("unlock.lua"));
    UNLOCK_SCRIPT.setResultType(Long.class);
}
```

- `static final` + `static` 块：类加载时执行一次，全局一份
- `setLocation(new ClassPathResource("unlock.lua"))`：从 classpath 根目录读脚本（文件放 `src/main/resources/`）
- `setResultType(Long.class)`：Lua 的 `return 0` 和 `del` 的返回值都是整数，告诉 Spring 转成 `Long`

不能每次读文件：读文件 + 编译有开销，锁是秒杀链路上最热的方法。`DefaultRedisScript` 内部缓存 SHA1，后续用 `EVALSHA` 只传摘要不传脚本内容，省带宽。

### 3.3 tryLock：value 存什么

```java
@Override
public boolean tryLock(long timeoutSec) {
    String threadId = ID_PREFIX + Thread.currentThread().getId();
    Boolean success = stringRedisTemplate.opsForValue()
            .setIfAbsent(KEY_PREFIX + name, threadId, timeoutSec, TimeUnit.SECONDS);
    return Boolean.TRUE.equals(success);   // 防空指针
}
```

和之前比只改一处：`threadId` 由 `long` 变 `String`（前缀 + 线程 id），因为要和 Lua 里的 `ARGV[1]` 做字符串比较。`setIfAbsent(key, value, timeout, unit)` = `SET key value NX EX timeout`。

### 3.4 unLock：KEYS 和 ARGV 怎么传

```java
@Override
public void unLock() {
    stringRedisTemplate.execute(
            UNLOCK_SCRIPT,                                  // 脚本
            Collections.singletonList(KEY_PREFIX + name),   // KEYS
            ID_PREFIX + Thread.currentThread().getId()      // ARGV
    );
}
```

| 位置 | 传什么 | Lua 里怎么取 |
| --- | --- | --- |
| 第二个参数（List） | 所有 key | `KEYS[1]`、`KEYS[2]`… |
| 第三个参数起（可变参数） | 其他参数 | `ARGV[1]`、`ARGV[2]`… |

Lua 下标**从 1 开始**，不是 0。`Collections.singletonList(...)` 是只有一个元素的不可变 List，用来传单个 key。

---

## 四、unlock.lua 逐行解析

```lua
-- 比较线程标示与锁中的标示是否一致
if(redis.call('get',KEYS[1]) == ARGV[1]) then
    --释放锁 del key
    return redis.call('del',KEYS[1])
end
return 0
```

- `redis.call('get', KEYS[1])`：取锁的 value（谁持有）
- `== ARGV[1]`：和「我」的标识比，相等说明锁是我的
- `return redis.call('del', KEYS[1])`：删掉，`del` 返回删除数量（1 表示删成功）
- `return 0`：不是我的锁，什么都不做

返回 `Long`：`1` 表示确实删了，`0` 表示没删（锁已不属于自己）。

---

## 五、面试考点

### Q1 释放锁为什么要用 Lua 脚本，不用 Java 判断？

> Java 的 `get → equals → delete` 三步之间可能被打断：A 判断通过后锁恰好 TTL 到期自动释放，B 抢到锁，A 再 delete 就误删了 B 的锁。Redis 执行 Lua 期间不会插入其他命令，把「比较 + 删除」变成不可分割的整体。原子性的本质是**执行期间不被打断**，不是把多步变一步。

### Q2 锁的 value 为什么是「UUID + 线程 id」？

> 释放前要判断锁是不是自己的，所以 value 必须能唯一标识持有者。线程 id 只在单个 JVM 内唯一，集群下两台机器都有线程 37，会误判。加 JVM 级 UUID 前缀后变成全局唯一。口诀：**本地唯一 + 进程唯一 = 全局唯一**。

### Q3 `execute` 的参数怎么传？KEYS 和 ARGV 有什么区别？

> 第二个参数（List）传所有 key，对应 Lua 的 `KEYS[1]…`；第三个参数起（可变参数）传其他数据，对应 `ARGV[1]…`。Lua 下标从 1 开始。约定 key 和非 key 分开传，是为了兼容 Redis Cluster（key 才能路由到正确节点）。

### Q4 `DefaultRedisScript` 为什么用 static 块初始化？

> 脚本对象全局只需一份：类加载时执行一次，避免每次调用都读文件、编译。内部会缓存脚本 SHA1，后续走 `EVALSHA` 只传摘要，省带宽。

### Q5 手写 SETNX 锁还有什么不足？【诚实答】

> ①不可重入——同线程二次获取会失败（Redisson 用 Hash 记录重入次数解决）；②业务超过 TTL 锁提前释放——看门狗自动续期；③拿不到锁只能循环重试，无法阻塞唤醒（Redisson 用 pub/sub）。生产上会用 Redisson，手写是为了理解原理。

### Q6 `SET NX EX` 的三个特性分别靠什么？

> 互斥靠 `NX`（只有一个能设置成功）；防死锁靠 `EX`（持锁方宕机锁也会到期释放）；高可用靠 Redis 集群。

---

## 六、今日重点

### 6.1 必记 5 件事

1. **Lua 解决的问题**：释放锁时「判断 + 删除」之间有空隙，会误删别人的锁。
2. **value = UUID + 线程 id**：本地唯一 + 进程唯一 = 全局唯一。
3. **execute 传参**：第二个参数是 KEYS，可变参数是 ARGV，Lua 下标从 1 开始。
4. **脚本只加载一次**：static 块 + SHA1 缓存走 `EVALSHA`。
5. **原子性的准确含义**：多步在执行期间不被打断。

### 6.2 代码速记

```java
// 抢锁：SET key value NX EX
Boolean success = stringRedisTemplate.opsForValue()
        .setIfAbsent("lock:" + name, ID_PREFIX + Thread.currentThread().getId(),
                     timeoutSec, TimeUnit.SECONDS);

// 释放：Lua 保证「判断 + 删除」原子
stringRedisTemplate.execute(UNLOCK_SCRIPT,
        Collections.singletonList(KEY_PREFIX + name),
        ID_PREFIX + Thread.currentThread().getId());
```

```lua
if(redis.call('get',KEYS[1]) == ARGV[1]) then
    return redis.call('del',KEYS[1])
end
return 0
```

### 6.3 易错点清单

| ❌ 易错 | ✅ 正确 |
| --- | --- |
| 释放锁直接 `delete` | 先判断标识再删，且判断+删除放 Lua |
| value 只存线程 id | 前面加 JVM 级 UUID，防跨实例误删 |
| `long` 直接塞 value | 转 String（`+ ""` 或 `String.valueOf`） |
| Lua 下标写 0 | 从 1 开始（`KEYS[1]`、`ARGV[1]`） |
| 把业务参数塞进 KEYS | KEYS 只放 key，其余走 ARGV |
| 每次调用都 new 脚本 | `static final` + static 块，全局一份 |
| `setIfAbsent` 返回值直接拆箱 | `Boolean.TRUE.equals(flag)` 防 NPE |

### 6.4 待补

| 知识点 | 状态 |
| --- | --- |
| Redisson（可重入、看门狗、锁重试） | 待学 |
| 秒杀优化（Lua 判断资格 + 异步下单） | 本周目标，学完更新简历第 5 条 |
| Redis 消息队列（List / PubSub / Stream） | 待学 |
| 达人探店 / 好友关注 / GEO / 签到 / UV | 只看视频，不敲 |
