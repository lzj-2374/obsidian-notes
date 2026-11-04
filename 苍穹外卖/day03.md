# Day03 笔记

> [!info] 项目
> 苍穹外卖 · 第 3 天
>
> 涉及内容：`@RequestBody` 使用规则、Redis（数据结构 + 常用命令）
>
> 上一篇：[[苍穹外卖/day02]]

- **tags**: #苍穹外卖 #SpringBoot #Redis

---

## 1. @RequestBody 使用规则

- 前端返回数据形式是 **JSON** 的话，Controller 层传入的实体对象就要加 `@RequestBody`。
- 前端返回的数据格式是 **Query**（在浏览器地址后面输入 `?xx` 就能访问），就不需要加 `@RequestBody`。

---

## 2. Redis

Redis 是一种基于**内存**的 key-value 结构的数据库：

- 基于内存存储，读写性能高
- 适合存储一些热点数据（热点商品、资讯、新闻）

### 2.1 启动与连接

在 Redis 文件目录下：

1. 启动服务：打开 cmd 窗口，输入 `redis-server.exe redis.windows.conf`，回车
2. 连接客户端：新开一个 cmd 窗口，输入 `redis-cli.exe`，回车
   - 输入 `keys *` 回车，查看所有 key
   - 输入 `exit` 退出
   - 也可以用 `redis-cli.exe -h localhost -p 6379` 指定连接的主机和端口

### 2.2 5 种常用数据类型

Redis 存储的是 key-value 结构的数据，其中 key 是字符串类型，value 有 5 种常用的数据类型：

| 类型 | 说明 |
|---|---|
| 字符串 string | 普通字符串，Redis 最简单的数据类型（比较常用） |
| 哈希 hash | 也叫散列，类似于 Java 中的 HashMap 结构 |
| 列表 list | 按照插入顺序排序，可以有重复元素，类似于 Java 中的 LinkedList |
| 集合 set | 无序集合，没有重复元素，类似于 Java 中的 HashSet |
| 有序集合 sorted set / zset | 集合中每一个元素关联一个分数（score），根据分数升序排序，没有重复元素 |

### 2.3 字符串操作命令

| 命令 | 说明 |
|---|---|
| `SET key value` | 设置指定 key 的值 |
| `GET key` | 获取指定 key 的值 |
| `SETEX key seconds value` | 设置指定 key 的值，并将 key 的过期时间设为 seconds 秒 |
| `SETNX key value` | 只有在 key 不存在时设置 key 的值 |

### 2.4 哈希操作命令

Redis hash 是一个 string 类型的 field 和 value 的映射表，hash 特别适合用于存储对象。常用命令：

| 命令 | 说明 |
|---|---|
| `HSET key field value` | 将哈希表 key 中的字段 field 的值设为 value |
| `HGET key field` | 获取存储在哈希表中指定字段的值 |
| `HDEL key field` | 删除存储在哈希表中的指定字段 |
| `HKEYS key` | 获取哈希表中所有字段 |
| `HVALS key` | 获取哈希表中所有值 |

### 2.5 列表操作命令

Redis 列表是简单的字符串列表，按照插入顺序排序。常用命令：

| 命令 | 说明 |
|---|---|
| `LPUSH key value1 [value2]` | 将一个或多个值插入到列表头部 |
| `LRANGE key start stop` | 获取列表指定范围内的元素 |
| `RPOP key` | 移除并获取列表最后一个元素 |
| `LLEN key` | 获取列表长度 |

### 2.6 集合操作命令

Redis set 是 string 类型的无序集合，集合成员是唯一的，集合中不能出现重复的数据。常用命令：

| 命令 | 说明 |
|---|---|
| `SADD key member1 [member2]` | 向集合添加一个或多个成员 |
| `SMEMBERS key` | 返回集合中的所有成员 |
| `SCARD key` | 获取集合的成员数 |
| `SINTER key1 [key2]` | 返回给定所有集合的交集 |
| `SUNION key1 [key2]` | 返回所有给定集合的并集 |
| `SREM key member1 [member2]` | 删除集合中一个或多个成员 |

### 2.7 有序集合操作命令

Redis 有序集合是 string 类型元素的集合，且不允许有重复成员。每个元素都会关联一个 double 类型的分数。常用命令：

| 命令 | 说明 |
|---|---|
| `ZADD key score1 member1 [score2 member2]` | 向有序集合添加一个或多个成员 |
| `ZRANGE key start stop [WITHSCORES]` | 通过索引区间返回有序集合中指定区间内的成员 |
| `ZINCRBY key increment member` | 有序集合中对指定成员的分数加上增量 increment |
| `ZREM key member [member ...]` | 移除有序集合中的一个或多个成员 |

### 2.8 Redis 通用命令

Redis 的通用命令**不分数据类型**，所有类型都适用：

| 命令 | 说明 |
|---|---|
| `KEYS pattern` | 查找所有符合给定模式（pattern）的 key |
| `EXISTS key` | 检查给定 key 是否存在 |
| `TYPE key` | 返回 key 所储存的值的类型 |
| `DEL key` | 该命令用于在 key 存在时删除 key |


---

## 3. 今日重点

> [!star] 重点回顾
>
> 1. **`@RequestBody` 使用规则**：JSON 请求加 `@RequestBody`，Query 参数不加。
> 2. **Redis 特点**：基于内存的 key-value 数据库，读写性能高，适合存储热点数据。
> 3. **5 种数据类型**：string、hash、list、set、zset（有序集合），各自特点要能区分。
> 4. **常用命令**：
>    - 字符串：SET / GET / SETEX / SETNX
>    - 哈希：HSET / HGET / HDEL / HKEYS / HVALS
>    - 列表：LPUSH / LRANGE / RPOP / LLEN
>    - 集合：SADD / SMEMBERS / SCARD / SINTER / SUNION / SREM
>    - 有序集合：ZADD / ZRANGE / ZINCRBY / ZREM
> 5. **通用命令**（不分数据类型）：`KEYS pattern`（按模式查找 key）、`EXISTS key`（判断是否存在）、`TYPE key`（查看值的类型）、`DEL key`（删除 key）。

---

## 4. 面试问题

> [!question] 🎤 面试题
> **问题**：Redis 有哪 5 种常用数据类型？分别适合什么场景？为什么 Redis 读写这么快？

**参考答案**：

1. **5 种类型及场景**：
   - **string**：最常用，缓存、计数器、session 存储；
   - **hash**：存对象（如用户信息），字段级更新，内存友好；
   - **list**：双向链表，消息队列、 Feed 流、栈/队列；
   - **set**：无序不重复，随机抽奖、去重、共同好友（交集）；
   - **zset**：带分数的有序集合，排行榜、延迟队列。
2. **为什么快**：
   - **基于内存**存储，无磁盘 IO；
   - **单线程模型**避免锁竞争与上下文切换（命令本身极快）；
   - **key-value 简单结构**，查找多为 O(1)；
   - 配合 IO 多路复用（epoll）支持海量连接。
