# Day08 · 一人一单与分布式锁

> **本日路线**：一人一单需求 → 查重条件 → 并发问题 → `synchronized` 锁 userId（锁与事务的顺序、代理对象、intern）→ 集群失效 → 分布式锁方案对比 → 手写 `SimpleRedisLock`
>
> **一句话**：一人一单是「查了再插」的竞态，必须加悲观锁；但锁必须包住事务、且要跨 JVM 可见，于是从 `synchronized` 走向基于 Redis `SET NX EX` 的分布式锁。

---

## 目录

- [[#一、一人一单的需求与流程]]
- [[#二、为什么必须两个条件一起查]]
- [[#三、一人一单的并发问题]]
- [[#四、用 synchronized 锁 userId]]
- [[#五、集群下 synchronized 为什么失效]]
- [[#六、分布式锁]]
- [[#七、SimpleRedisLock 实现细节]]
- [[#八、面试考点]]
- [[#九、今日重点]]

---

## 一、一人一单的需求与流程

**需求**：修改秒杀业务，要求同一个优惠券，一个用户只能下一单。

**流程**：

```
开始 → 提交优惠券id → 查询优惠券信息 → 判断秒杀是否开始
  → 未开始：返回异常，结束
  → 已开始：判断库存是否充足
      → 不充足：返回异常，结束
      → 充足：根据优惠券id + 用户id 查询订单
          → 订单存在：返回异常，结束
          → 订单不存在：扣减库存 → 创建订单 → 返回订单id → 结束
```

和昨天比，只多了一步「查订单判重」，但这一步引出了锁的问题。

---

## 二、为什么必须两个条件一起查

```java
int count = query().eq("user_id", userId).eq("voucher_id", voucherId).count();
if (count > 0) {
    return Result.fail("已经买过一次了");
}
```

「一人一单」限制的是**同一用户 + 同一张券**这个组合，单独一个条件范围都不对：

| 只用一个条件 | 查出来的范围 | 后果 |
| --- | --- | --- |
| 只有 `voucher_id` | 所有人买这张券的订单 | 第一个人下单后其他人全被拦 → 变成「一券一单」 |
| 只有 `user_id` | 该用户所有券的订单 | 买过 A 券就买不了 B 券 → 变成「一人一生一单」 |

对应 SQL：`WHERE voucher_id = ? AND user_id = ?`，缺一个语义就变了。

**小知识点**：MyBatis-Plus 3.4.3 的 `count()` 返回 `int`（3.5.x 改成 `long`），用 `int` 接最合适 —— `COUNT(*)` 永远有值，用 `Integer` 只会多一次装箱、多一层 NPE 顾虑。

---

## 三、一人一单的并发问题

`count` 查询和后面的 insert 不是原子的，同一用户的多个请求可以同时读到「没买过」：

```
线程A：查 count=0 → 放行 → 扣库存 → insert
线程B：查 count=0 → 放行 → 扣库存 → insert   → 同一用户同一张券 2 条订单
```

- **乐观锁救不了**：超卖能用 `stock > 0`，是因为那是 UPDATE，可以「更新时校验」；一人一单是 INSERT，插入前没有行可比较
- **`@Transactional` 也救不了**：事务只保证扣库存和建订单一起成功/失败，不管「查」和「插」之间被别人插队
- **解决方向**：用悲观锁按 `userId` 加锁，让同一用户的请求串行；锁粒度是 userId，不同用户不该互相阻塞

---

## 四、用 synchronized 锁 userId

```java
Long userId = UserHolder.getUser().getId();
synchronized (userId.toString().intern()) {
    IVoucherOrderService proxy = (IVoucherOrderService) AopContext.currentProxy();
    return proxy.createVoucherOrder(voucherId);
}
```

### 4.1 为什么锁不能加在方法上 / 方法里

核心：**锁的范围必须包住事务的范围**。

- `@Transactional` 是 Spring **代理**在方法外层开事务、方法返回后提交
- `synchronized` 是 JVM 在**代码块退出时**释放

所以 synchronized 写在 `createVoucherOrder` 里（方法上或方法内都一样），顺序必然是：

```
开事务 → 加锁 → 查订单 → 插入 → 释放锁 → 提交事务
                                    ↑ 订单还在未提交状态
```

锁释放时订单还没提交，别的线程抢到锁照样查不到 → 重复下单。

| 写法 | 结果 |
| --- | --- |
| 加在 `createVoucherOrder` 方法上 | ❌ 锁在事务里 + 锁的是 this，所有用户排队 |
| 加在 `createVoucherOrder` 方法内部 | ❌ 锁仍在事务里，释放早于提交 |
| 加在 `seckillVoucher` 方法上 | ⚠️ 正确但锁 this，所有用户全部串行，QPS 掉到单线程 |
| 外层无事务方法里加锁 + 代理调用 | ✅ 只锁同一用户，且锁包住提交 |

→ 只有「**外层无事务方法加锁 → 锁内调用代理的事务方法**」才对。锁的覆盖范围就变成「加锁 → 开事务 → 提交 → 释放锁」。

### 4.2 代理对象 AopContext.currentProxy()

`@Transactional` 生效的前提是**从代理对象进方法**。Spring 生成 CGLIB 子类代理，事务逻辑写在代理里：

```
代理（开事务）→ 目标对象方法体 → 代理（提交/回滚）
```

- `this.createVoucherOrder()` 是自调用，走原始对象，**绕过代理** → 事务失效
- `AopContext.currentProxy()` 取出当前线程正在用的代理对象，再调用才走代理 → 事务生效
- 前提：启动类要有 `@EnableAspectJAutoProxy(exposeProxy = true)`（本项目第 8 行已有），否则抛异常

### 4.3 intern() 是啥

`synchronized` 比的是「**是不是同一个对象**」，不是内容相等。

- `userId.toString()` 每次都 new 一个 String，两线程地址不同 → 不是同一把锁 → 锁失效
- `intern()` 去**字符串常量池**取相同内容的唯一引用 → 同一 userId 拿到同一个对象 → 锁得住

为什么不用 `synchronized (userId)` 直接锁 Long：`Long` 缓存只有 -128~127，超出范围每次装箱都是新对象，一样锁不住。

### 4.4 代码执行顺序

1. `UserHolder.getUser().getId()` 取当前用户
2. `userId.toString().intern()` 生成该用户专属锁对象
3. `synchronized` 同一用户的请求排队，不同用户互不影响
4. `AopContext.currentProxy()` 拿代理对象，为了能开事务
5. `proxy.createVoucherOrder()` 走代理 → 开事务 → 查订单/建订单 → 提交
6. 提交完退出 synchronized 释放锁，下一个同用户请求才进来

---

## 五、集群下 synchronized 为什么失效

`@Transactional` 加锁只解决了**单机**的一人一单问题，集群模式下就不行了。

**根本原因：锁的状态存在 JVM 自己的内存里**（对象头 Mark Word + Monitor），是进程私有内存。

```
同一用户的两个请求 → Nginx 分发
  → Tomcat A（JVM 1）：synchronized 加锁成功
  → Tomcat B（JVM 2）：synchronized 加锁成功   ← 各自锁的只是自己 JVM 里同名的对象
→ 两边都查到 count=0 → 都插入 → 订单表 2 条
```

一句话：**它只能保证同一个 JVM 内互斥，跨进程失效**；要跨机器，就得把锁放到 Redis 这种公共存储里 —— 这就是分布式锁。

判断标准：**只要锁的状态需要跨机器共享，就不能用 JVM 锁**（`synchronized`、`ReentrantLock` 都不行）。

---

## 六、分布式锁

**定义**：满足分布式系统或集群模式下**多线程可见并且互斥**的锁。

**特性**：

1. 多进程可见
2. 互斥
3. 高可用
4. 高性能
5. 安全性

### 6.1 三种常见实现对比

| | MySQL | Redis | Zookeeper |
| --- | --- | --- | --- |
| **互斥** | 利用 mysql 本身的互斥锁机制 | 利用 setnx 这样的互斥命令 | 利用节点的唯一性和有序性实现互斥 |
| **高可用** | 好 | 好 | 好 |
| **高性能** | 一般 | 好 | 一般 |
| **安全性** | 断开连接，自动释放锁 | 利用锁超时时间，到期释放 | 临时节点，断开连接自动释放 |

- 三种都能互斥，**Redis 胜在高性能 + 实现简单**，所以本项目用它
- Zookeeper 的临时节点安全性更稳（连接断了锁自动没），但性能一般、还要额外部署
- 记忆口诀：**互斥靠“谁先写成功”（MySQL 锁 / setnx / 节点唯一），安全靠“连接断了自动放”（MySQL、ZK）或“超时到期”（Redis）**

### 6.2 实现分布式锁的两个基本方法

**获取锁** —— 互斥（只能一个线程拿到）、非阻塞：

```
# 添加锁，NX 是互斥、EX 是设置超时时间
SET lock thread1 NX EX 10
```

**释放锁** —— 手动释放 或 超时释放（获取锁时带的那个超时时间）：

```
# 释放锁，删除即可
DEL key
```

对应 Java API：`stringRedisTemplate.opsForValue().setIfAbsent(key, value, timeout, unit)`。

---

## 七、SimpleRedisLock 实现细节

`ILock` 接口定义两个方法（`tryLock(timeoutSec)` / `unLock()`），`SimpleRedisLock` 用 Redis 实现：`tryLock` 走 `setIfAbsent`（= SET NX EX），`unLock` 走 `delete`。

### 7.1 为什么 stringRedisTemplate 不用 @Resource 注入

`SimpleRedisLock` **不是 Spring Bean**：类上没加 `@Component`，扫描不到 → Spring 不会创建它 → 字段注入也不生效（恒为 null）。

它是每次 `new` 出来的临时对象，`name` 是实例自己的状态；做成单例就变成所有用户共用一把锁，锁粒度变全局。所以依赖只能靠**构造器传**：Service 里注入 `StringRedisTemplate`，`new` 时传进去。

> 规律：`@Controller` / `@Service` 都是 `@Component` 的衍生注解，靠启动类所在包（`com.hmdp`）及子包的**组件扫描**收集；`@Mapper` 是 MyBatis 扫描后生成代理再注册。没注解的类，扫描器扫到也会跳过。

对照：`CacheClient` 无状态、要复用 → `@Component` + 构造器注入；锁对象有状态 → 必须 `new`。

### 7.2 key 为什么是 KEY_PREFIX + name

```java
private static final String KEY_PREFIX = "lock:";
// 调用侧：new SimpleRedisLock(stringRedisTemplate, "order:" + userId)
// 最终 key：lock:order:101
```

- `"lock:"` 是命名空间，防止和缓存 key（`cache:shop:1`）撞车，也方便排查/批量清理
- `name` 是业务标识，由调用方决定

分两段的设计意图：**工具类不认识具体业务**，只管往 Redis 抢一个 key，和 `CacheClient` 用 `keyPrefix` 是一个思路。

为什么 name 要带 `"order:"`：如果只传 `userId`，key 就是 `lock:101`，万一别处也拿 101 当锁（比如优惠券 id 也是 101）就撞了。

规律：**锁的 key 要能唯一标识“要保护的那个东西”**。

### 7.3 threadId 是 long，为什么 + "" 就能存

`StringRedisTemplate` 的 value 序列化器是 `StringRedisSerializer`，**只收字符串**，直接塞 long 会报类型不匹配。

`threadId + ""` 是字符串拼接：编译器把 long 转成 String 再拼空串，等价于 `String.valueOf(threadId)`，得到 `"37"`。

后面判断锁归属时，从 Redis 取出来的也是 String，同样要 `String.valueOf(...)` 转一次再比。

---

## 八、面试考点

### Q1：一人一单为什么要用「优惠券id + 用户id」两个条件查？

**答**：限制的是「同一用户 + 同一张券」这个组合。只按券查会变成一券全服一单，只按用户查会变成一人一生一单，都不对。

### Q2：一人一单为什么乐观锁解决不了？

**答**：乐观锁的前提是「更新时能校验旧值」，超卖能用 `stock > 0` 是因为那是 UPDATE。一人一单是 INSERT，插入前没有可比较的行，只能用悲观锁让它串行。

### Q3：为什么 `synchronized` 不能加在 `@Transactional` 方法上？

**答**：`@Transactional` 由代理在方法外层开事务、方法返回后提交；`synchronized` 在方法体结束就释放。顺序是「释放锁 → 提交事务」，锁释放时订单还未提交，其他线程查库看不到，仍会重复下单。正确做法是把锁加在无事务的外层方法，锁内通过代理调用事务方法。

### Q4：`AopContext.currentProxy()` 是干什么的？

**答**：`this.xxx()` 是自调用，走原始对象，绕过 Spring 代理，`@Transactional` 失效。`currentProxy()` 拿到当前线程使用的代理对象，通过它调用才会被事务拦截器增强。前提是 `@EnableAspectJAutoProxy(exposeProxy = true)`。

### Q5：为什么锁对象用 `userId.toString().intern()`，不直接锁 `userId`？

**答**：`synchronized` 比的是对象地址。`Long` 只有 -128~127 有缓存，超出范围每次装箱都是新对象，锁不住；`toString()` 也每次 new String。`intern()` 让相同内容从字符串常量池返回同一个引用，才能锁住同一用户。

### Q6：集群下 `synchronized` 为什么失效？怎么解决？

**答**：锁状态在 JVM 自己的内存（Mark Word + Monitor），是多进程私有的。集群有多台机器多个 JVM，各自加锁都成功；加上 Nginx 会把同一用户的两个请求分到不同机器，照样重复下单。解决：把锁状态放到公共存储，用分布式锁（Redis `SET NX EX` / Zookeeper 临时节点）。

### Q7：分布式锁有哪些实现？为什么选 Redis？

**答**：MySQL（行锁 / 唯一索引）、Redis（`SETNX`）、Zookeeper（临时有序节点）。都能互斥，Redis 胜在高性能、实现简单、项目里本来就有；Zookeeper 连接断开自动释放更安全，但性能一般且需额外部署。

### Q8（思考）：`unLock` 直接 `DEL` 有什么风险？

**答**：可能删掉别人的锁。业务执行超时 → 锁到期自动释放 → 别的线程拿到锁 → 原线程执行完执行 `DEL`，把别人的锁删了，锁形同虚设。所以释放前要判断 value 是不是自己的线程标识，且「判断 + 删除」要保证原子性（下一步用 Lua 解决）。

---

## 九、今日重点

### 9.1 必记 5 件事

1. **一人一单的钥匙**：`user_id` + `voucher_id` 两个条件一起查，缺一个语义就错。
2. **锁必须包住事务**：外层无事务方法加锁 → 锁内通过**代理**调用事务方法。
3. **`this` 自调用绕过代理**：`AopContext.currentProxy()` 才能让 `@Transactional` 生效（要开 `exposeProxy`）。
4. **锁对象要唯一**：`userId.toString().intern()`；`Long` 缓存只有 -128~127，直接锁 Long 不可靠。
5. **JVM 锁跨进程无效**：集群要上分布式锁，Redis 用 `SET NX EX`。

### 9.2 代码 / API 速记

```java
// 单机版：锁包住事务
synchronized (userId.toString().intern()) {
    IVoucherOrderService proxy = (IVoucherOrderService) AopContext.currentProxy();
    return proxy.createVoucherOrder(voucherId);
}

// 分布式锁：抢锁（SET NX EX）
Boolean success = stringRedisTemplate.opsForValue()
        .setIfAbsent("lock:" + name, String.valueOf(threadId), timeoutSec, TimeUnit.SECONDS);
return Boolean.TRUE.equals(success);

// 释放锁
stringRedisTemplate.delete("lock:" + name);

// 查重
int count = query().eq("user_id", userId).eq("voucher_id", voucherId).count();
```

### 9.3 易错点清单

| ❌ 易错 | ✅ 正确 |
| --- | --- |
| 只按券 id 或只按用户 id 查重 | 两个条件 `AND` 一起查 |
| `synchronized` 加在 `@Transactional` 方法上 | 锁加在无事务的外层方法，锁内调代理 |
| `this.createVoucherOrder()` 调事务方法 | `AopContext.currentProxy().createVoucherOrder()` |
| `synchronized (userId)` 锁 Long | `userId.toString().intern()` |
| 以为加了 `synchronized` 集群也安全 | 集群要分布式锁 |
| `setIfAbsent` 返回值直接拆箱 | `Boolean.TRUE.equals(flag)` 防 NPE |
| 给 `SimpleRedisLock` 加 `@Component` | 它是 new 出来的有状态对象，加了就成单例锁全局 |
| 锁 value 存 long | `StringRedisTemplate` 只收 String，用 `+ ""` 或 `String.valueOf` |

### 9.4 待补

| 知识点 | 状态 |
| --- | --- |
| 锁误删问题 + Lua 脚本保证原子性 | 待学（当前 `unLock` 直接 DEL 有风险） |
| Redisson 分布式锁 | 待学（可重入、锁重试、看门狗续期） |
| Redis 优化秒杀 | 待学（Lua 判断资格 + 异步下单） |
| Redis 消息队列实现异步秒杀 | 待学 |
