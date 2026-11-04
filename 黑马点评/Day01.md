# Day01 · Redis 快速入门

> **一句话概括**：Redis 是一个**基于内存的键值型 NoSQL 数据库**，单线程执行命令保证原子性，靠"内存 + IO 多路复用 + 高效编码"做到极低延迟，同时支持持久化、主从、分片，是黑马点评项目里缓存、秒杀、点赞、排行榜的核心中间件。

---

## 目录

- [[#一、Redis 是什么]]
- [[#二、SQL 与 NoSQL 的对比]]
- [[#三、Linux 环境搭建与常用命令]]
- [[#四、Redis 通用命令]]
- [[#五、Key 的层级命名规范]]
- [[#六、五大基本数据类型]]
- [[#七、面试常问（高频真题）]]
- [[#八、今日重点]]

---

## 一、Redis 是什么

**定位**：键值型（Key-Value）数据库、NoSQL、远程词典服务器（通过网络访问的内存数据字典）。

### 1.1 核心特征

| 特征 | 说明 |
| --- | --- |
| 键值型 | value 支持多种数据结构，功能丰富 |
| 单线程 | 命令串行执行，**每个命令都具备原子性**（线程安全） |
| 低延迟 | 基于内存（核心原因）、IO 多路复用、良好的编码 |
| 持久化 | 支持 RDB / AOF，断电数据不会消失 |
| 高可用与扩展 | 支持主从集群、分片集群 |
| 生态 | 支持多语言客户端 |

### 1.2 为什么单线程还这么快？

1. **纯内存操作**（最核心原因，不走磁盘 IO）
2. **IO 多路复用**（epoll，单线程处理大量连接）
3. **单线程避免线程切换与锁竞争的开销**
4. **精心设计的数据结构与编码**（如 String 的 SDS、ZSet 的跳表）

> 记忆口诀：**"内存 + 复用 + 无锁 + 好结构"**。

---

## 二、SQL 与 NoSQL 的对比

| SQL（关系型） | NoSQL（非关系型） | 对比维度 |
| --- | --- | --- |
| 结构化 | 非结构化 | 数据结构 |
| 关联的（表之间有关联） | 无关联 | 数据关联 |
| SQL 查询 | 非 SQL 命令 | 查询方式 |
| ACID（四大特性） | BASE（基本满足） | 事务特性 |
| 磁盘 | 内存（读取快） | 存储方式 |
| 垂直扩展（加硬件） | 水平扩展（加节点） | 扩展性 |
| ① 数据结构固定<br>② 对数据安全性、一致性要求较高 | ① 数据结构不固定<br>② 对一致性、安全性要求不高<br>③ 对性能要求高 | 使用场景 |

---

## 三、Linux 环境搭建与常用命令

已安装在虚拟机 `/usr/local/src` 目录下。

### 3.1 基础操作

```bash
# 打开配置文件
sudo nano /usr/local/src/redis-6.2.6/redis.conf

# 前台启动 Redis（指定配置文件）
redis-server /usr/local/src/redis-6.2.6/redis.conf

# 查看服务是否启动
ps -ef | grep redis

# 直接启动（使用默认配置）
redis-server

# 进入cli操作页面
redis-cli

# 通过客户端优雅关闭
redis-cli shutdown

# 查看配置项含义（万能命令）
redis-cli help <command>
```

```bash
# 杀死进程
kill 4484          # 发送 SIGTERM，优雅退出
kill -9 4484       # 发送 SIGKILL，强制杀死（可能丢数据）

# 前台运行时退出：CTRL + C
```

### 3.2 systemd 常用命令（背下来）

```bash
# 查看状态
sudo systemctl status redis
# 启动
sudo systemctl start redis
# 停止
sudo systemctl stop redis
# 重启
sudo systemctl restart redis
# 设置开机自启（已执行过）
sudo systemctl enable redis
# 取消开机自启
sudo systemctl disable redis
```

### 3.3 ⚠️ 踩坑记录：daemonize 必须为 no

源码编译安装 Redis，用 systemd 托管时，**`redis.conf` 中的 `daemonize` 必须改成 `no`**。

- 如果 `daemonize = yes`：Redis 自己 fork 到后台运行，主进程退出，systemd 判定服务已结束 → 服务直接死掉，状态变成 `inactive (dead)`。
- 原理：systemd 用 `Type=forking` 还是 `Type=simple` 判断进程，前台模式（no）最省心。

```conf
# /usr/local/src/redis-6.2.6/redis.conf
daemonize no
```

---

## 四、Redis 通用命令

> 不知道某个命令什么意思？直接 `help 命令名`。

| 命令 | 作用 | 备注 |
| --- | --- | --- |
| `KEYS pattern` | 查看符合模板的所有 key | ⚠️ 生产环境禁用，全量遍历会**阻塞主线程** |
| `DEL key [key ...]` | 删除一个或多个 key | 支持批量 |
| `EXISTS key` | 判断 key 是否存在 | 存在返回 1，不存在返回 0 |
| `EXPIRE key seconds` | 给 key 设置有效期（秒） | 到期自动删除 |
| `TTL key` | 查看 key 剩余有效期 | **-1 = 永久，-2 = 不存在** |

```bash
127.0.0.1:6379> set name jack ex 10
OK
127.0.0.1:6379> ttl name
(integer) 10
127.0.0.1:6379> ttl notExistKey
(integer) -2
```

**小结论**：`TTL` 的两个特殊返回值是面试 / 笔试题常考细节 —— `-1` 是"活着但没设过期"，`-2` 是"已经死了"。

---

## 五、Key 的层级命名规范

Redis 的 key 允许多个单词用 `:` 分隔形成层级结构，便于区分业务：

```
项目名:业务名:类型:id
```

格式不固定，可按需增删词条。例如项目叫 `heima`，有 user 和 product 两类数据：

```bash
# 用户 1 号
heima:user:1
# 商品 1 号
heima:product:1
```

好处：在客户端（如 RedisInsight）里会展示成树形目录，方便管理和排查。

---

## 六、五大基本数据类型

### 6.1 String 类型

最基础的存储类型，value 是字符串。按内容格式可分三类：

| 分类 | 说明 | 能力 |
| --- | --- | --- |
| `string` | 普通字符串 | 普通读写 |
| `int` | 整数 | 可自增、自减 |
| `float` | 浮点数 | 可自增、自减 |

> 底层都是**字节数组**存储，只是编码方式不同；**单个 value 最大 512MB**。

#### 常用命令

| 命令 | 作用 |
| --- | --- |
| `SET key value` | 添加或修改键值对 |
| `GET key` | 根据 key 获取 value |
| `MSET k1 v1 k2 v2 ...` | 批量添加 |
| `MGET k1 k2 ...` | 批量获取 |
| `INCR key` | 整型自增 1 |
| `INCRBY key n` | 整型自增指定步长 |
| `INCRBYFLOAT key n` | 浮点型自增指定步长 |
| `SETNX key value` | key **不存在**时才添加（等价 `SET key value NX`） |
| `SETEX key seconds value` | 添加并指定有效期（等价 `SET key value EX 10`） |

```bash
SET num 10              # OK
INCR num                # 11
INCRBY num 5            # 16
INCRBYFLOAT num 1.5     # 17.5

SETNX lock 1            # 第一次返回 1（成功）
SETNX lock 2            # 第二次返回 0（失败，未覆盖）

SET k1 v1 EX 10         # 带过期时间的一条命令搞定
```

#### 典型应用场景

计数器（阅读量、点赞数）、分布式锁（`SETNX` / `SET NX EX`）、Session 共享、对象缓存（JSON 序列化存储）。

---

### 6.2 Hash 类型

也叫散列，value 是一个无序字典，类似 Java 的 `HashMap`。

**结构**：一个大 key 内部包含多个 `field-value`：

```
key: heima:user:1
  ├── field: name → value: Jack
  └── field: age  → value: 21
```

**优点**：对象每个字段**独立存储**，可以针对单个字段做 CRUD，不用操作整个对象。

> 对比 String 存对象：String 修改任意字段都要 **读完整 JSON → 改字段 → 重新序列化 → 覆盖写入**，既浪费带宽又可能并发覆盖。

#### 常用命令

| 命令 | 作用 |
| --- | --- |
| `HSET key field value` | 添加 / 修改指定 field 的值 |
| `HGET key field` | 获取单个 field 的值 |
| `HMSET` | 批量添加（Redis 4.0 后 `HSET` 支持多参数，已逐渐被替代） |
| `HMGET key f1 f2` | 批量获取多个 field |
| `HGETALL key` | 获取**全部 field 和 value** |
| `HKEYS key` | 只获取所有 field（字段名） |
| `HVALS key` | 只获取所有 value（字段值） |
| `HINCRBY key field n` | 对某个 field 做数值自增，可指定步长 |
| `HSETNX key field value` | 仅当 **field 不存在**时才添加，不覆盖旧值 |

```bash
HSET heima:user:1 name Jack age 21
HGET heima:user:1 name        # "Jack"
HINCRBY heima:user:1 age 1    # 22
HGETALL heima:user:1
# 1) "name"  2) "Jack"  3) "age"  4) "22"
```

**易错点**：

- `HSETNX` 是**字段级别**的 nx，不是 key 级别。
- `HGETALL` 一次性取出全部数据，**field 数量很大时要慎用**，容易阻塞主线程。

---

### 6.3 List 类型

类似 Java 的 `LinkedList`，可看作**双向链表**，支持正向和反向检索。

**特征**（与 LinkedList 一致）：

- 有序
- 元素可以重复
- 插入和删除快
- 查询速度一般（中间索引慢）

**常用场景**：有序数据，如朋友圈点赞列表、评论列表、简易消息队列。

#### 常用命令

| 命令 | 作用 |
| --- | --- |
| `LPUSH key element ...` | 向**左侧**插入一个或多个元素 |
| `LPOP key` | 移除并返回**左侧**第一个元素；空列表返回 `nil` |
| `RPUSH key element ...` | 向**右侧**插入一个或多个元素 |
| `RPOP key` | 移除并返回**右侧**第一个元素 |
| `LRANGE key start end` | 返回下标区间内的元素，`end=-1` 代表取全部 |
| `BLPOP / BRPOP` | 阻塞版 LPOP / RPOP，无元素时**等待指定时长**而非立即返回 nil |

```bash
RPUSH queue a b c        # 列表: a b c
LRANGE queue 0 -1        # 1) "a" 2) "b" 3) "c"
LPOP queue               # "a"

# 阻塞 3 秒等待元素，可用于简易消息队列
BLPOP queue 3
```

#### ★ 经典思考题：如何用 List 模拟栈 / 队列 / 阻塞队列

| 目标 | 存 | 取 | 说明 |
| --- | --- | --- | --- |
| **栈**（后进先出） | `LPUSH` | `LPOP` | 出口和入口在**同一边** |
| | `RPUSH` | `RPOP` | 同上 |
| **队列**（先进先出） | `LPUSH` | `RPOP` | 入口和出口**不在同一边** |
| | `RPUSH` | `LPOP` | 同上 |
| **阻塞队列** | 任一侧 | 出队用 `BLPOP` / `BRPOP` | 入口出口不同边 + 阻塞弹出 |

> 这题是**面试高频**，答的时候一定要说清"同一边 vs 不同边"的逻辑。

---

### 6.4 Set 类型

类似 Java 的 `HashSet`，可看作 **value 为 null 的 HashMap**（本质是 hash 表）。

**特征**：

- 无序
- 元素不可重复
- 查找快
- 支持交集、并集、差集

**常用场景**：去重、共同好友、共同关注、抽奖（`SPOP`）、标签。

#### 常用命令

| 命令 | 作用 |
| --- | --- |
| `SADD key member ...` | 添加元素，重复元素不会存入 |
| `SREM key member ...` | 移除指定元素 |
| `SCARD key` | 统计元素总个数 |
| `SISMEMBER key member` | 判断元素是否存在，存在返回 1，否则 0 |
| `SMEMBERS key` | 获取全部元素 |

#### 集合运算

| 命令 | 含义 |
| --- | --- |
| `SINTER key1 key2 ...` | 交集：多个集合**共同存在**的元素 |
| `SUNION key1 key2 ...` | 并集：合并并自动去重 |
| `SDIFF key1 key2 ...` | 差集：`key1` 有、后面集合没有的元素 |

```bash
SADD s1 a b c
SADD s2 b c d
SINTER s1 s2     # b c  （共同好友）
SUNION s1 s2     # a b c d
SDIFF s1 s2      # a  （我关注了他没关注 = 可能认识的人）
```

---

### 6.5 SortedSet 类型（ZSet）

一个**可排序的 Set**，类似 Java 的 `TreeSet`，但底层差别很大：

> **底层实现 = 跳表（SkipList） + hash 表**。每个元素带 `score` 属性，按 score 排序。

**特征**：

- 可排序
- 元素不重复
- 查询速度快

**常用场景**：排行榜（点赞榜、销量榜、热搜榜）。

#### 基础命令

| 命令 | 作用 |
| --- | --- |
| `ZADD key score member` | 添加元素；member 已存在则**更新 score** |
| `ZREM key member` | 删除指定 member |
| `ZSCORE key member` | 查询 member 的 score |
| `ZRANK key member` | 获取排名（默认升序，从 0 开始） |
| `ZCARD key` | 元素总数量 |
| `ZCOUNT key min max` | 统计 score 在 `[min, max]` 区间的元素个数 |
| `ZINCRBY key increment member` | 给 member 的 score 增加步长（可为负数） |
| `ZRANGE key min max` | 按 score 升序，取**排名区间**内元素 |
| `ZRANGEBYSCORE key min max` | 按 score 升序，取**分值区间**内元素 |

#### 集合运算

`ZINTER`（交）、`ZUNION`（并）、`ZDIFF`（差）

```bash
ZADD rank 100 tom 90 jerry 80 lucy
ZINCRBY rank 10 lucy         # lucy 分数变 90
ZRANGE rank 0 -1 WITHSCORES  # 升序输出
ZREVRANGE rank 0 2           # 降序取前三名（排行榜）
```

> **注意**：所有排名默认升序；要降序就在 `Z` 后面加 `REV`，如 `ZREVRANK`、`ZREVRANGE`、`ZREVRANGEBYSCORE`。

---

### 6.6 五大类型速查对比

| 类型 | 底层结构 | 有序 | 可重复 | 典型场景 |
| --- | --- | --- | --- | --- |
| String | SDS（字节数组） | - | - | 缓存对象、计数器、分布式锁 |
| Hash | hash 表 | 无序（field） | field 不重复 | 对象字段局部更新、购物车 |
| List | 双向链表（quicklist） | ✅ | ✅ | 消息队列、点赞/评论列表 |
| Set | hash 表 | ❌ | ❌ | 去重、共同好友、抽奖 |
| SortedSet | 跳表 + hash 表 | ✅（按 score） | ❌ | 排行榜、延时队列 |

---

## 七、面试常问（高频真题）

> 全部来自**真实面试高频考点**，且都能用本日笔记的知识点回答。

### Q1：Redis 为什么这么快？单线程为什么还能这么快？

**答**：四个原因 ——

1. **纯内存操作**：数据在内存里，读写不走磁盘 IO，这是最核心原因；
2. **IO 多路复用**：用 epoll 单线程管理大量连接，避免为每个连接创建线程；
3. **单线程无锁无切换**：不存在多线程竞争和上下文切换开销，同时保证命令原子性；
4. **高效的数据结构**：SDS、跳表、ziplist / listpack 等经过极致优化。

> 追问："单线程"指什么？—— 指**命令执行**是单线程的；Redis 6.0 之后网络 IO 读写是多线程的，但核心命令仍串行执行。

### Q2：Redis 是单线程还是多线程？

**答**：**命令处理是单线程**（所以每个命令天然原子），Redis 6.0 引入了**多线程 IO** 只用于网络数据的读写和协议解析，命令执行依旧是主线程串行，因此不会破坏原子性。

### Q3：SQL 和 NoSQL 有什么区别？什么时候用 Redis 而不是 MySQL？

**答**：从**数据结构、数据关联、查询方式、事务特性、存储介质、扩展方式、使用场景**七个维度对比（见 [[#二、SQL 与 NoSQL 的对比]]）。

选型结论：

- 数据结构固定、对**安全性 / 一致性**要求高、需要复杂关联查询 → **MySQL**；
- 数据结构不固定、对**性能**要求高、一致性要求可放宽 → **Redis**。

两者通常是**配合**使用：MySQL 做持久化底库，Redis 做缓存层。

### Q4：五种数据类型分别用在什么场景？（必问）

| 类型 | 场景举例 |
| --- | --- |
| String | 对象缓存（JSON）、阅读量计数（`INCR`）、分布式锁（`SETNX`） |
| Hash | 用户信息、购物车（field = 商品 id，value = 数量） |
| List | 消息队列、朋友圈点赞/评论列表、最新 N 条动态 |
| Set | 点赞去重、共同好友（`SINTER`）、抽奖（`SPOP`） |
| SortedSet | 排行榜、热搜榜、延时队列（score 存时间戳） |

### Q5：如何用 Redis 的 List 实现栈、队列、阻塞队列？

**答**：

- **栈**（同一边进出）：`LPUSH + LPOP`，或 `RPUSH + RPOP`；
- **队列**（不同边进出）：`LPUSH + RPOP`，或 `RPUSH + LPOP`；
- **阻塞队列**：入口出口不同边，出队用 `BLPOP` / `BRPOP`，队列为空时阻塞等待，而不是空轮询返回 nil。

```bash
# 队列
LPUSH msgQueue task1
RPOP  msgQueue

# 阻塞队列：最多等 5 秒
BRPOP msgQueue 5
```

### Q6：SortedSet 底层为什么用跳表，而不是红黑树 / 平衡树？

**答**（这题能答出来很加分）：

1. **范围查询更简单高效**：跳表底层是有序链表，`ZRANGEBYSCORE` 这类区间查询只需找到起点后顺序遍历，红黑树还要做中序遍历；
2. **实现和维护简单**：跳表不需要旋转、变色等复杂平衡操作，代码量小、易调试；
3. **并发 / 插入删除性能稳定**：跳表插入删除只影响局部指针，平均 O(logN)，且更容易做无锁实现；
4. 用**跳表 + hash 表**组合，既能用 hash 表 O(1) 查到 member 的 score，又能用跳表按 score 排序。

### Q7：Redis 的 key 为什么设过期时间？`TTL` 返回 -1 和 -2 分别代表什么？

**答**：设过期时间是为了**控制内存占用、避免冷数据长期驻留**（也用于验证码、Session、分布式锁自动释放）。

- `-1`：key 存在，但**没有设置过期时间**（永久有效）；
- `-2`：key **不存在**（可能从来没设过，也可能已过期被删）。

### Q8：`SETNX` 有什么用？

**答**：**只有 key 不存在时才写入**，天然具备"抢占"语义，是实现**分布式锁**的基础：

```bash
# 加锁：key 不存在才成功，同时设置过期时间防死锁（原子操作）
SET lock:order:1001 1 NX EX 10

# 解锁（实际项目要用 Lua 脚本保证"判断 + 删除"原子性）
DEL lock:order:1001
```

> 加分点：`SETNX` 单独用 + `EXPIRE` 是**两步非原子**操作，中间宕机会死锁，所以必须用 `SET key value NX EX` 一条命令。

### Q9：用 String 存对象和用 Hash 存对象有什么区别？

**答**：

- **String**：整个对象序列化成 JSON。优点是结构简单、一次取全；缺点是**改一个字段也要全量读写**，还有并发覆盖风险。
- **Hash**：字段独立存储，可 `HINCRBY` 局部更新。优点是节省带宽、支持字段级操作；缺点是字段过多时 `HGETALL` 会阻塞，且不方便做整体序列化传输。

> 选择：**需要频繁改单个字段 → Hash；整体读多写少 / 需要缓存整个 JSON → String**。

### Q10：为什么生产环境禁止使用 `KEYS *`？

**答**：`KEYS` 会**全库遍历**所有 key，时间复杂度 O(N)，而 Redis 命令是**单线程串行执行**的，遍历期间会**阻塞其他所有请求**，key 数量大时直接导致服务雪崩。

替代方案：用 `SCAN cursor MATCH pattern COUNT n` **游标分批遍历**，每次只返回少量元素，不会长时间阻塞。

```bash
SCAN 0 MATCH heima:user:* COUNT 100
```

### Q11：Redis 的 value 最大能存多大？

**答**：String 类型单个 value 最大 **512MB**。实际业务中不会存这么大，因为大 key 会造成网络阻塞、删除时阻塞主线程。

### Q12（运维/部署常问）：源码编译安装 + systemd 托管时，为什么要设 `daemonize no`？

**答**：`daemonize yes` 会让 Redis fork 到后台，主进程退出，systemd 认为服务已结束，状态变 `inactive (dead)`。设为 `no` 让 Redis 前台运行，由 systemd 接管进程生命周期，这样才能正常 `start / stop / restart` 和开机自启。

---

## 八、今日重点

### 8.1 必须记住的 5 件事

1. **Redis 快的四个原因**：内存 / IO 多路复用 / 单线程无锁 / 高效数据结构 —— 面试第一问，必须张口就来。
2. **五大类型的底层与场景**：String（缓存、计数）、Hash（对象字段）、List（队列、列表）、Set（去重、交并差）、ZSet（排行榜，**跳表 + hash**）。
3. **List 模拟栈 / 队列 / 阻塞队列**：同边进出 = 栈，不同边 = 队列，`BLPOP` = 阻塞队列。
4. **`SETNX` / `SET NX EX` = 分布式锁的雏形**：这是后面秒杀、一人一单的基石。
5. **`TTL` 的 -1 与 -2**、**`KEYS` 生产禁用（用 `SCAN`）** 这两个细节，是区分"背过"和"懂了"的分水岭。

### 8.2 命令速记表

```bash
# ---------- 通用 ----------
KEYS pattern        EXISTS key         DEL key
EXPIRE key 10       TTL key

# ---------- String ----------
SET / GET / MSET / MGET
INCR / INCRBY / INCRBYFLOAT
SETNX key v         SET key v EX 10

# ---------- Hash ----------
HSET / HGET / HMSET / HMGET
HGETALL / HKEYS / HVALS
HINCRBY key field n     HSETNX key field v

# ---------- List ----------
LPUSH / LPOP / RPUSH / RPOP
LRANGE key 0 -1     BLPOP / BRPOP

# ---------- Set ----------
SADD / SREM / SCARD / SISMEMBER / SMEMBERS
SINTER / SUNION / SDIFF

# ---------- SortedSet ----------
ZADD / ZREM / ZSCORE / ZRANK / ZCARD / ZCOUNT
ZINCRBY key n member
ZRANGE key 0 -1     ZRANGEBYSCORE key min max
ZREVRANGE / ZREVRANK   ← 降序加 REV
```

### 8.3 易错点清单（复习时只看这里）

| ❌ 易错 | ✅ 正确 |
| --- | --- |
| 认为 Redis 完全单线程 | 命令执行单线程，6.0+ 网络 IO 多线程 |
| `TTL` 返回 -1 当成"已过期" | -1 = 永久，**-2 = 不存在** |
| `HSETNX` 当成 key 级判断 | 是 **field 级** 的 nx |
| `KEYS *` 图方便在线上用 | 用 `SCAN` 分批遍历 |
| 认为 ZSet 底层是红黑树 | 是 **跳表 + hash 表** |
| `SETNX` + `EXPIRE` 两步加锁 | 一条 `SET k v NX EX 10` 保证原子 |
| systemd 托管还设 `daemonize yes` | 必须 `daemonize no` |

### 8.4 明日预告

- 缓存相关的实战：缓存穿透 / 击穿 / 雪崩怎么用 Redis 解决
- 黑马点评里 Redis 的真正用法：**短信登录（Redis 共享 Session）、商户缓存、秒杀一人一单（`SETNX` + Lua）、点赞（Set）、排行榜（ZSet GEO）**
