# Day06 笔记

> [!info] 项目
> 苍穹外卖 · 第 6 天
>
> 涉及内容：Redis 缓存菜品、Spring Cache 注解、Lombok @Builder、购物车、地址簿
>
> 上一篇：[[苍穹外卖/day05]]

- **tags**: #苍穹外卖 #SpringBoot #Redis #SpringCache

---

## 1. Redis 缓存菜品数据

通过 Redis 缓存菜品数据，减少数据库查询操作。

### 1.1 缓存流程

开始查询菜品，请求到后端服务，首先去缓存中判断该菜品数据是否存在：

- 如果**存在**，直接读取缓存数据，返回给小程序；
- 如果**不存在**，则去数据库查询，并添加到缓存里面，以便下次查询。

### 1.2 缓存逻辑分析

- 每个分类下的菜品保存一份缓存数据
- 数据库中菜品数据有变更时清理缓存数据（新增菜品、修改菜品、批量删除菜品、起售、停售菜品都要清理缓存数据，不清理的话，小程序展示的和管理端的不一致，修改管理端接口 `DishController` 的相关方法，加入清理缓存逻辑）

### 1.3 生效原理：B端修改菜品，C端为何立马生效

> [!tip] 核心：Cache Aside（旁路缓存）模式
> **不是"实时推送"，而是「B端改完删缓存，C端下次查询回源」**。读的时候走缓存，写的时候删缓存。
> "立马生效"的准确含义：缓存一删，C端后续任何一次请求都不可能再读到旧数据；又因为缓存在 Redis 集中存放，多个后端节点共享，删一次就是全局生效。

**C端读**（`sky-server/.../controller/user/DishController.java`）：

1. 构造 key：`dish_ + 分类id`（C端永远按分类查菜品，缓存的就是"分类 → 菜品列表"这个映射）
2. 先查 Redis：命中且 `size() > 0` → 直接返回，不打数据库
3. 未命中 → 查 MySQL（该分类 + 起售状态，含口味）→ 写回 Redis → 返回

**B端写**（`sky-server/.../controller/admin/DishController.java`）：

1. 新增 / 修改 / 批量删除 / 起售停售 → **先写 MySQL**
2. 紧接着 `cleanCache("dish_*")`：用 `keys` 找到所有菜品缓存 key，一并 `delete`
3. C端下一次请求必然未命中 → 回源查 MySQL（已是新数据）→ 用户"立马"看到新菜品

```java
/**
 * 根据条件清理缓存
 */
private void cleanCache(String pattern) {
    Set keys = redisTemplate.keys(pattern);
    redisTemplate.delete(keys);
}
```

#### 1.3.1 四个关键设计点

1. **删缓存，而不是更新缓存**：① B端要把整个 `DishVO`（含口味列表）重建再写回，成本高；② 两个B端请求并发时可能互相覆盖成旧数据；③ 删除是幂等的，最安全
2. **为什么用 `dish_*` 通配删，而不是 `dish_ + 菜品id`**：缓存的 key 是**分类维度**，B端修改时手里只有菜品id，反推不出该删哪个 key（代码里注释掉的 `dish_ + dishDTO.getId()` 就是这个思路，行不通），所以按前缀全删，宁可错杀
3. **顺序不能反：先写库、后删缓存**。反了会有"缓存删了 → C端回源读到旧库 → 旧数据又回填进缓存 → 缓存长期脏"的窗口
4. **仍有极小不一致窗口**：C端"读库 → 写缓存"之间，B端恰好改库+删缓存，可能把旧数据写回缓存。课程项目可忽略；工业级方案用**延迟双删**、或订阅 **binlog（Canal）** 异步删缓存

#### 1.3.2 工程注意点

- `redisTemplate.keys("dish_*")` 是 O(N) 的**阻塞命令**，生产环境 key 空间大时要慎用，可改为维护 key 集合或删具体 key
- 当前写缓存时**没设过期时间**，完全靠B端清理保证一致，建议加个 **TTL 兜底**，万一漏删缓存不会永久脏
- `RedisConfiguration` 只配了 key 的 String 序列化，value 走默认 JDK 序列化，所以 redis-cli 里能看到 `dish_xxx` 的 key 但 value 是二进制乱码；想让缓存可读，给 value 配 `GenericJackson2JsonRedisSerializer`
- 套餐是另一套缓存（`SetmealController` 里 `cleanCache("setmeal_*")`），但**改菜品不会动套餐缓存**——套餐自身数据没变，只是引用了菜品

---

## 2. Spring Cache

Spring Cache 是一个框架，实现了基于注解的缓存功能，只需要简单加一个注解，就能实现缓存功能。

Spring Cache 提供了一层抽象，底层可以切换不同的缓存实现，如：

- **EhCache**
- **Caffeine**
- **Redis**

### 2.1 导入坐标

```xml
<!-- Spring Cache 核心starter -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

### 2.2 常用注解

> 前提：启动类加 `@EnableCaching` 开启缓存

#### 2.2.1 @Cacheable

**作用**：查询，有缓存直接读缓存；没有就执行方法，把结果存入缓存

```java
@Cacheable(value = "user", key = "#id")
public User getUserById(Long id){
    //查数据库
}
```

- `value/cacheNames`：缓存组名称（缓存的命名空间）
- `key`：缓存 key，支持 SpEL 表达式
- `condition`：**满足条件才缓存**，`condition = "#id>0"`
- `unless`：满足条件**不缓存结果**，`unless="#result == null"` 结果为 null 不缓存

#### 2.2.2 @CachePut

**作用**：新增 / 修改，一定会执行方法，把返回值更新到缓存，不会查缓存

```java
@CachePut(cacheNames = "user", key = "#user.id")
public User updateUser(User user){
    //更新数据库
}
```

> 场景：更新数据，同步刷新缓存，方法必须返回对象

#### 2.2.3 @CacheEvict

**作用**：删除缓存，清除指定 key 缓存

```java
//删除指定key缓存
@CacheEvict(cacheNames = "user", key = "#id")
public void deleteUser(Long id){}

//allEntries = true：清空该cacheNames下全部缓存
@CacheEvict(cacheNames = "user", allEntries = true)

//beforeInvocation = true：方法执行之前删缓存，默认false方法成功后删
@CacheEvict(beforeInvocation = true)
```

- `allEntries=true`：清除该缓存名下所有 key，慎用
- `beforeInvocation=true`：方法抛异常也会删除缓存

#### 2.2.4 @Caching

**作用**：组合多个缓存注解，一个方法同时配置多个缓存操作

```java
@Caching(
    cacheable = {@Cacheable(value="user",key="#id")},
    put = {@CachePut(value="user",key="#result.id")},
    evict = {@CacheEvict(value="userList",allEntries=true)}
)
public User test(Long id){}
```

#### 2.2.5 @CacheConfig

**作用**：类级别注解，抽取公共配置，类下所有方法复用

```java
@CacheConfig(cacheNames = "user") //统一指定cacheNames，方法不用重复写
public class UserService {
    @Cacheable(key="#id") //不用写cacheNames
    public User getUser(Long id){}
}
```

### 2.3 SpEL 表达式常用 key 写法（面试常考）

| 写法 | 含义 |
|---|---|
| `#id` | 取方法参数 id |
| `#user.id` | 参数对象 user 的 id 属性 |
| `#result.id` | 方法返回对象的 id |
| `#root.args[0]` | 第一个参数 |
| `#root.methodName` | 当前方法名 |

### 2.4 面试速记背诵

1. **@Cacheable**：先查缓存，没有执行方法再存缓存（查询用）
2. **@CachePut**：一定执行方法，更新缓存（新增修改）
3. **@CacheEvict**：删除缓存（删除数据）
4. **@Caching**：组合多个缓存注解
5. **@CacheConfig**：类上统一提取公共 cacheNames

---

## 3. 购物车功能

### 3.1 添加购物车逻辑

要先判断当前商品是否在购物车中：

- 如果**存在**，只需执行一个修改操作，将数量加一
- 如果**不存在**，执行 insert 插入操作
- 不同的用户需要有自己的购物车（根据 `user_id` 这个字段区分），也就是说，在查询购物车商品时，要将 `user_id` 作为条件去查询

### 3.2 查看购物车代码

`ShoppingCartServiceImpl`：

```java
@Override
public List<ShoppingCart> showShoppingCart() {
    Long userId = BaseContext.getCurrentId();
    ShoppingCart shoppingCart = ShoppingCart.builder()
            .userId(userId)
            .build();
    List<ShoppingCart> list = shoppingCartMapper.list(shoppingCart);
    return list;
}
```

### 3.3 @Builder 注解说明

`@Builder` 注解让 Lombok 在编译时自动生成一整套"构建"代码，其中就包括这个静态方法 `ShoppingCart.builder()`，返回一个 Builder 构建器对象。

**【这样写等价于】**

```java
ShoppingCart shoppingCart = new ShoppingCart();
shoppingCart.setUserId(userId);
```

即：新建对象 → 设置 userId → 其他字段保持 null。

**【为什么用 Builder 而不是 new + set】**

可读性好、链式书写：

```java
ShoppingCart.builder()
        .userId(userId)
        .build();
```

中间可以按需继续链式设值，比如：

```java
ShoppingCart.builder()
        .userId(userId)
        .status(StatusConstant.ENABLE)
        .build();   // 调用 build() 才真正创建对象
```

适合"字段多但只想设少数几个"的场景——这里的意图就是：构造一个只带 userId 的查询条件对象，去查"当前用户的所有购物车记录"（后面大概是 `shoppingCartMapper.list(shoppingCart)`）。和之前 C 端 `DishController` 里 `new Dish()` + 两个 setter 再当查询条件用，是同一个套路，只是写法换了个风格。

---

## 4. 导入地址簿功能

### 4.1 业务功能

- 查询地址列表
- 新增地址
- 修改地址
- 删除地址
- 设置默认地址
- 查询默认地址（在下单页面会把它查询出来）

### 4.2 参考资料

在 `D:\BaiduNetdiskDownload\苍穹外卖资料\资料\day08\地址簿模块功能代码` 目录下，后续关于地址功能可直接使用。

---

## 5. 今日重点

> [!star] 重点回顾
>
> 1. **缓存菜品数据**：先查缓存 → 命中则直接返回；未命中则查数据库 → 写入缓存。菜品数据有变更时（新增/修改/删除/起售/停售）要清理缓存。
> 2. **Spring Cache 五大注解**：
>    - `@Cacheable`（查询，先查缓存）
>    - `@CachePut`（更新，一定执行方法，刷新缓存）
>    - `@CacheEvict`（删除缓存）
>    - `@Caching`（组合多个缓存注解）
>    - `@CacheConfig`（类级别，提取公共 cacheNames）
> 3. **SpEL 表达式**：`#id`（方法参数）、`#user.id`（对象属性）、`#result.id`（返回值属性）。
> 4. **购物车逻辑**：根据 `user_id` 区分用户购物车，查询时把 `user_id` 作为条件。
> 5. **@Builder**：Lombok 生成的构建器模式，链式书写，适合"字段多但只设少数几个"的场景。
> 6. **地址簿功能**：查询列表、新增、修改、删除、设置默认、查询默认（下单页用到）。

---

## 6. 面试问题

> [!question] 🎤 面试题
> **问题**：Spring Cache 五大注解分别是什么、各适用什么场景？菜品缓存里"数据库变了但缓存没清"会造成什么问题，如何避免？

**参考答案**：

1. **五大注解**：
   - **@Cacheable**：查询用，先查缓存、命中直接返回；未命中执行方法并把结果存缓存；
   - **@CachePut**：新增/修改用，**一定执行方法**并把返回值更新进缓存（不查缓存），方法必须返回对象；
   - **@CacheEvict**：删除/变更用，清除指定 key（`allEntries=true` 清空整个 cacheNames；`beforeInvocation=true` 方法执行前删）；
   - **@Caching**：组合多个缓存注解在一个方法上；
   - **@CacheConfig**：类级别，统一提取公共 cacheNames。
2. **"库变了缓存没清"的问题**：
   - 缓存与数据库**不一致**，导致小程序展示与管理端数据不一致（如改了菜名/价格，前端还显示旧的）。
3. **如何避免**：
   - 对菜品的**新增、修改、批量删除、起售、停售**都要清理对应缓存（优先按 key 清，`allEntries` 慎用）；
   - 注意顺序：**先改数据库、再清缓存**；一致性要求更高时可用**延迟双删**、或把缓存设较短的过期时间兜底；
   - 并发场景下也可考虑"先清缓存 + 延迟双删"或订阅 binlog 等方案。
