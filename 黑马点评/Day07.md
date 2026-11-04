# Day07 · 全局唯一 ID 与优惠券秒杀

> **本日路线**：全局唯一 ID（RedisIdWorker）→ 优惠券秒杀下单 → 超卖问题（乐观锁 / 悲观锁）→ 踩坑两个（零日期、NaN）
>
> **一句话**：用「秒级时间戳 << 32 | 当天 INCR 序号」拼出全局唯一订单 ID；秒杀下单先跑通流程，再用乐观锁（`stock > 0`）拦住超卖，一人一单和分布式锁留到下一步。

---

## 目录

- [[#一、全局唯一 ID]]
- [[#二、优惠券秒杀下单]]
- [[#三、超卖问题]]
- [[#四、踩坑记录]]
- [[#五、面试考点]]
- [[#六、今日重点]]

---

## 一、全局唯一 ID

每个店铺都可以发布优惠券，当用户抢购时，就会生成订单并保存到 `tb_voucher_order` 这张表中。订单表如果使用数据库自增 id 会存在一系列问题：

1. id 的规律性太明显
2. 受单表数据量的限制

全局 ID 生成器，是一种分布式系统下用来生成全局唯一 ID 的工具，一般要满足以下特性：

1. 唯一性
2. 高可用
3. 高性能
4. 递增性
5. 安全性

### 1.1 代码实现：RedisIdWorker

```java
@Component
public class RedisIdWorker {
    // 起始时间：2026-10-24 00:00:00 UTC
    private static final long BEGIN_TIMESTAMP = 1792800000;
    // 序列号占 32 位
    private static final int COUNT_BITS = 32;

    public long nextId(String keyPrefix) {
        // 1. 时间戳 = 当前秒 - 起始秒
        LocalDateTime now = LocalDateTime.now();
        long nowSecond = now.toEpochSecond(ZoneOffset.UTC);
        long timestamp = nowSecond - BEGIN_TIMESTAMP;

        // 2. 序列号 = 当天第几个（Redis INCR，原子自增）
        String date = now.format(DateTimeFormatter.ofPattern("yyyy:MM:dd"));
        Long count = stringRedisTemplate.opsForValue()
                .increment("icr:" + keyPrefix + ":" + date);

        // 3. 拼接：时间戳放高位，序列号放低位
        return timestamp << COUNT_BITS | count;
    }
}
```

三步：**算时间戳 → 拿当天序号 → 移位拼接**。

- 时间戳：从自定义起始时间算起，从 0 开始，省位数
- 序列号：每天一个 key，`INCR` 原子自增，每天从 1 重新开始
- 拼接：`timestamp << 32 | count`

### 1.2 64 位怎么分配

```
0 | 时间戳（31 bit）| 序列号（32 bit）
↑
符号位（1 bit，恒为 0）
```

`1 + 31 + 32 = 64`，正好一个 long。

### 1.3 为什么用左移 + 或，不能直接拼

1. **返回类型是 long**：`timestamp + "" + count` 得到的是 String，类型不对。
2. **字符串拼接没有固定宽度，会撞**：
   - `timestamp = 1, count = 23` → `"123"`
   - `timestamp = 12, count = 3` → `"123"`（撞了）
3. **移位 + 或是固定宽度的二进制拼接**：
   - `timestamp << 32`：把时间戳挪到高位，低 32 位补 0，64 位是固定的，所以右边多余部分会自动去除
   - `| count`：把序号填进低 32 位，两段互不重叠，不会进位

一句话：**左移负责腾位置，按位或负责填序号。**

### 1.4 压测结果（300 线程 × 100 个 ID）

- 生成 30000 个 ID，**0 重复**，耗时 4119ms，QPS ≈ 7283
- 瓶颈不在算法，在 Lettuce 连接池 `max-active: 10`

---

## 二、优惠券秒杀下单

每个店铺都可以发布优惠券，分为平价券和特价券。平价券可以任意购买，特价券要秒杀抢购。

表关系如下：

- `tb_voucher`：优惠券的基本信息，优惠金额、使用规则等
- `tb_seckill_voucher`：优惠券的库存、开始抢购时间、结束抢购时间。特价优惠券才需要填写这些信息。

下单时要判断两点：

1. 秒杀是否开始或结束，如果尚未开始或已经结束则无法下单
2. 库存是否充足，不足无法下单

流程：

```
开始 --> 提交优惠券id --> 查询优惠券信息 --> 判断秒杀是否开始
  --> 未开始返回异常结果结束
  --> 开始 --> 判断库存是否充足 --> 不充足返回异常结果结束
  --> 充足扣减库存 --> 创建订单 --> 返回订单id --> 结束
```

### 2.1 代码流程（VoucherOrderServiceImpl.seckillVoucher）

查优惠券 → 判开始时间 → 判结束时间 → 判库存 → 扣库存 → 创建订单 → 返回订单号

### 2.2 知识点

1. **秒杀三道校验**：开始时间、结束时间、库存
2. **`setSql("stock=stock-1")`**：让数据库直接做减法，避免"先查再改"
3. **`@Transactional`**：扣库存和建订单必须同时成功或同时失败
4. **`RedisIdWorker.nextId("order")`**：生成全局唯一订单号
5. **`UserHolder.getUser().getId()`**：从 ThreadLocal 拿当前登录用户
6. **`@PathVariable`**：`/voucher-order/seckill/{id}` 绑定路径参数

### 2.3 两个待解决的问题

1. **超卖**：判断库存和扣库存不是原子的，而且 `stock=stock-1` 没带 `stock > 0` 条件，`update()` 只能拦住"券不存在"，拦不住"库存不够" → 见 [[#三、超卖问题]]
2. **一人一单**：没有校验同一用户是否已经购买过 → 待学

---

## 三、超卖问题

发生原因：假设库存是 1，线程 A 先查询库存，查询结果是 1，这时线程 B 也进来了，也执行查询库存操作，查询结果是 1，这时候线程 A 执行判断库存是否大于 1，是，扣减，剩余数量为 0；但是线程 B 查询的结果也是 1，也扣减了，剩余数量就变为 -1 了。

超卖问题是典型的多线程安全问题，最常见的解决方案就是加锁，分为乐观锁和悲观锁。

### 3.1 悲观锁

认为线程安全问题**一定会发生**，因此在数据操作之前先获取锁，确保线程串行执行。例如 `synchronized`、`Lock` 都属于悲观锁。

常见实现：

1. Java：`synchronized`、`ReentrantLock`
2. 数据库：`SELECT ... FOR UPDATE`（行锁）
3. 分布式：Redis `SETNX`、Redisson

两个坑：

1. `synchronized` 只在单机有效：集群下每个 JVM 各有一把锁，锁不住别的实例，必须用分布式锁
2. 锁和事务的顺序：`@Transactional` 方法里加 `synchronized`，锁释放时事务可能还没提交，别的线程会读到旧数据；正确做法是把锁加在事务外层

### 3.2 乐观锁

认为线程安全问题**不一定会发生**，因此不加锁，只是在更新数据库时去判断有没有其他线程对数据做了修改。

- 如果没有修改则认为是安全的，自己才更新数据
- 如果已经被其它线程修改，说明发生了安全问题，此时可以重试或异常

和悲观锁的区别：

- 悲观锁：先加锁、线程排队，冲突多时稳定但慢
- 乐观锁：更新时校验，冲突少时性能好，冲突多会大量失败重试

乐观锁的关键是判断之前查询得到的数据是否被修改过，常见方式有两种：

### 3.3 版本号法

表里加一个 `version` 字段：

| id | stock | version |
| --- | --- | --- |
| 1 | 100 | 1 |

两个线程同时来扣库存：

```
线程A：读到 stock=100, version=1
线程B：读到 stock=100, version=1

线程A：UPDATE ... SET stock=99, version=2 WHERE id=1 AND version=1
       → 影响 1 行，成功

线程B：UPDATE ... SET stock=99, version=2 WHERE id=1 AND version=1
       → 影响 0 行（version 已经是 2 了），失败
```

关键在 `WHERE version = 读到的旧版本`：

- 改到 1 行 → 期间没人改过，成功
- 改到 0 行 → 被别人抢先了，失败（重试或直接返回）

版本号每次 +1，还能避免 ABA 问题（值改回去但版本已经变了）。

### 3.4 CAS 法（本项目用的）

CAS = Compare And Swap：更新时比较字段的当前值，是否还是我读到的值。

`tb_seckill_voucher` 没有 version 字段，课程直接用 **stock 本身当比较值**：

```java
boolean success = seckillVoucherService.update()
        .setSql("stock = stock - 1")
        .eq("voucher_id", voucherId)
        .gt("stock", 0)        // ← 相当于版本条件
        .update();
```

- `stock > 0` 就是"版本条件"
- 两个线程同时扣，只有一个能把 stock 从 1 改成 0
- 另一个 `stock > 0` 不成立，影响 0 行，`update()` 返回 false
- 所以 `if (!success) return Result.fail("库存不足")` 才能真的拦住超卖
- 注意：**乐观锁只能防超卖，解决不了一人一单**

---

## 四、踩坑记录

### 4.1 查询优惠券时报「零日期」错误

**现象**：打开店铺详情，页面提示「服务器异常」，控制台报：

```
java.sql.SQLException: Value '0000-00-00 00:00:00' can not be represented as java.sql.Timestamp
```

**原因**：`tb_seckill_voucher` 表的 `begin_time` / `end_time` 默认值就是 `'0000-00-00 00:00:00'`。插入秒杀券时没指定这两个字段，就会存成零日期。MySQL 驱动没法把它转成 `LocalDateTime`，于是抛异常。

**解决**：把那条数据的时间改成有效值：

```sql
UPDATE tb_seckill_voucher
SET begin_time = '2026-10-01 00:00:00',
    end_time   = '2026-12-31 23:59:59'
WHERE voucher_id = 10;
```

以后插入秒杀券时，必须显式给 `begin_time` / `end_time`。

### 4.2 商铺详情页显示 NaN / 数据全空

**现象**：店铺详情页店名显示 `NaN`，评分、图片、地址、营业时间全部为空。

**排查**：直接请求接口：

```bash
curl http://localhost:8081/shop/1
# {"success":true}
```

返回里没有 `data`，说明 `Shop` 是 null。

**原因**：`queryById` 走的是逻辑过期 `cacheClient.queryWithLogicalExpire`，这个方法在缓存不存在时**直接返回 null，不查库**。而 `cache:shop:1` 没有预热，所以查不到 → 返回 null → 前端拿不到数据。

**解决**：先预热商铺缓存。

1. 把 `ShopServiceImpl` 第 233 行的 `saveShop2Redis` 从 `private` 改成 `public`
2. 测试类里加预热方法（注入 `ShopServiceImpl`，调用 `saveShop2Redis(1L, 20L)`）
3. 跑一次测试，刷新页面

**结论**：逻辑过期的前提是缓存已预热；缓存不存在时不会自动查库。

---

## 五、面试考点

### Q1：为什么订单 ID 不用数据库自增，要自己生成全局唯一 ID？

**答**：自增 ID 规律性太明显（容易被爬虫推算订单量），且受单表数据量限制，分库分表后会重复。全局 ID 生成器要满足唯一性、高可用、高性能、递增性、安全性。

### Q2：RedisIdWorker 的 64 位是怎么分配的？为什么这样设计？

**答**：`1 bit 符号位（恒 0）+ 31 bit 秒级时间戳 + 32 bit 序列号`。时间戳放高位保证整体趋势递增；序列号用 Redis `INCR` 按天生成（key 带 `yyyy:MM:dd`），既方便按天统计，也避免单 key 无限膨胀；每天从 1 重新开始，32 位单日足够用（约 42.9 亿）。

### Q3：为什么拼接用 `timestamp << 32 | count`，不用字符串拼接？

**答**：返回类型要求是 long，字符串拼接类型不对；且字符串拼接没有固定宽度会撞（`1+23` 和 `12+3` 都得 `"123"`）。左移是固定宽度的二进制拼接：`<< 32` 把时间戳挪到高位（long 只有 64 位，高位多余部分自动截断），`|` 把序号填进低 32 位，两段互不重叠、不会进位。

### Q4：`stock = stock - 1` 不加条件为什么会超卖？怎么解决？

**答**：判断库存和扣库存不是原子操作。线程 A、B 同时读到库存 1，都判断通过，都执行扣减，库存变 -1。解决：给 UPDATE 加上乐观锁条件 `.gt("stock", 0)`（CAS 法），更新时校验库存是否还大于 0，只有一个线程能扣成功，另一个影响 0 行返回 false。

### Q5：乐观锁和悲观锁的区别？各自适用什么场景？

**答**：悲观锁认为问题一定发生，操作前先加锁串行执行（`synchronized`、`SELECT ... FOR UPDATE`），冲突多时稳定但慢；乐观锁认为问题不一定发生，不加锁，更新时校验数据有没有被改过（版本号法、CAS），冲突少时性能好，冲突多会大量失败重试。

### Q6：版本号法和 CAS 法有什么区别？

**答**：版本号法在表里加 `version` 字段，`WHERE version = 读到的旧版本`，每次更新 version+1，还能避免 ABA 问题；CAS 法不额外加字段，直接拿业务字段本身当比较值（本项目用 `stock > 0`），实现更简单，但只能判断"还有没有库存"，语义上弱于版本号。

### Q7：`@Transactional` 方法里加 `synchronized` 有什么坑？

**答**：锁释放时事务可能还没提交，别的线程拿到锁后会读到旧数据。正确做法是把锁加在事务外层（外层方法加锁，内层方法才是事务）。另外 `synchronized` 只在单机有效，集群下每个 JVM 各有一把锁，必须换分布式锁。

---

## 六、今日重点

### 6.1 必记 5 件事

1. **ID 三步**：算时间戳 → `INCR` 拿当天序号 → `timestamp << 32 | count`。
2. **位运算口诀**：左移腾位置，按位或填序号；long 是 64 位定长，字符串拼接会撞。
3. **秒杀三道校验**：开始时间、结束时间、库存。
4. **防超卖一行代码**：`.gt("stock", 0)`（CAS 乐观锁），影响 0 行就返回失败。
5. **乐观锁的边界**：只能防超卖，解决不了一人一单。

### 6.2 代码 / API 速记

```java
// 全局唯一 ID
return timestamp << COUNT_BITS | count;   // 时间戳高位 + 序列号低位

// 序列号：按天 INCR
Long count = stringRedisTemplate.opsForValue()
        .increment("icr:" + keyPrefix + ":" + date);

// 扣库存（CAS 乐观锁防超卖）
boolean success = seckillVoucherService.update()
        .setSql("stock = stock - 1")
        .eq("voucher_id", voucherId)
        .gt("stock", 0)
        .update();
if (!success) return Result.fail("库存不足");
```

### 6.3 易错点清单

| ❌ 易错 | ✅ 正确 |
| --- | --- |
| 订单 ID 用数据库自增 | 全局唯一 ID（不暴露规律、支持分表） |
| 用字符串拼接生成 ID | `<< 32 \|` 二进制拼接（定长、不撞） |
| `update()` 只判了券存不存在 | 必须加 `.gt("stock", 0)` 拦超卖 |
| 以为乐观锁能同时解决一人一单 | 乐观锁只防超卖，一人一单要查订单表 |
| 零日期插入秒杀券不报错直到查询才炸 | 插入时必须显式给 `begin_time` / `end_time` |
| 逻辑过期不预热就上线查询 | 先预热，缓存不存在时它不查库、返回 null |
| `@Transactional` 里直接加 `synchronized` | 锁加在事务外层 |

### 6.4 待补

| 知识点 | 状态 |
| --- | --- |
| 一人一单 | 待学（查订单表 + 乐观锁/悲观锁校验） |
| 分布式锁 | 待学（`SETNX` 手写锁的缺陷 → Redisson） |
| Redis 优化秒杀 | 待学（Lua 脚本判断资格 + 异步下单） |
| Redis 消息队列实现异步秒杀 | 待学 |
