# Day02 笔记

> [!info] 项目
> 苍穹外卖 · 第 2 天
>
> 涉及内容：RESTful 设计规范、DTO 核心价值、时间格式处理、敏感字段脱敏、公共字段自动填充（AOP）
>
> 上一篇：[[苍穹外卖/day01]]

- **tags**: #苍穹外卖 #SpringBoot #DTO #AOP #RESTful

---

## 1. RESTful 接口设计规范

| 操作 | 方法 |
|---|---|
| 添加 | POST |
| 查询 | GET |
| 修改 | PUT |

---

## 2. DTO（数据传输对象）

### 2.1 为什么要使用 DTO？（核心价值）

1. **安全隔离与防止恶意注入**
   - 如果直接使用数据库实体类（Entity）接收前端参数，黑客可以通过抓包强行添加 `id`、`role`、`balance` 等敏感字段。若后端直接入库，会导致数据被恶意篡改（如越权修改他人资料、给自己充值）。
   - DTO 只声明允许前端传递的字段，天然屏蔽了非法字段的注入。
2. **结构解耦（前后端与数据库解耦）**
   - 数据库表结构（Entity）不应直接暴露给前端。若某天数据库字段改名（如 `username` 改为 `login_name`），如果不使用 DTO，前端的接口传参也必须跟着改。
   - 使用 DTO 后，Service 层负责将 DTO 转换为 Entity。无论数据库怎么变，只要前端传参不变，API 契约就是稳定的。
3. **数据裁剪与聚合**
   - **裁剪**：前端提交的表单往往不包含数据库自动生成的字段（如 `create_time`、`update_time`）或后端生成的字段（如加密后的密码）。
   - **聚合**：一个请求可能包含多个表的数据。例如"提交订单"，前端需要传订单信息、收货地址、商品列表。单一的 Entity 无法接收，必须定义一个聚合 DTO。
4. **独立的参数校验逻辑**
   - 同一个 Entity 可能在不同的业务场景下被复用（如新增、修改、查询）。如果在 Entity 上加 `@NotNull` 等校验注解，会导致逻辑冲突（新增需要校验，修改不需要）。
   - DTO 针对不同业务场景独立定义，可以在其属性上自由添加 `@Pattern`、`@Size` 等校验注解。
5. **避免"过度获取"与性能优化**
   - 实体类可能包含大字段（如 `text` 类型的文章内容、大对象）。如果前端只请求列表，直接把整个 Entity 查出来返回，会消耗大量带宽和内存。
   - DTO（或专用的返回对象）可以只包含列表所需的精简字段。

> [!example] 理解
> 拿 Employee 来说：如果前端传递过来的数据与 Employee 里的属性不一致（少传或多传），直接接收就可能出现 `null` 或脏数据。使用 `EmployeeDTO` 明确声明字段，才能避免这类问题。

### 2.2 DTO 在分层架构中的位置

- **Controller 层**：接收 HTTP 请求，使用 DTO 作为入参，并进行基础的参数校验（`@Valid`）。
- **Service 层**：接收 DTO，执行核心业务逻辑。在准备落库时，将 DTO 转换为 Entity。
- **Mapper/DAO 层**：只接收 Entity，负责与数据库交互。
- **数据返回**：查询数据库得到 Entity 后，Service 层将其转换为 VO（View Object）或 Response DTO，返回给 Controller。

---

## 3. 分页查询的时间格式处理

分页查询时，涉及到的时间字段（如最后修改时间）需要处理时间格式，有两种办法：

1. **在实体类字段上加 `@JsonFormat` 注解**（需要导包）：

   ```java
   @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
   private LocalDateTime createTime;
   ```

2. **扩展 SpringMVC 框架的消息转化器**：在配置类 `WebMvcConfiguration` 里加：

   ```java
   /**
    * 扩展Spring MVC框架的消息转化器
    * @param converters
    */
   @Override
   protected void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
       log.info("扩展消息转化器。。。");
       //创建一个消息转化器对象
       MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter();
       //需要为消息转化器设置一个对象转换器，对象转化器可以将Java对象序列化为json数据
       converter.setObjectMapper(new JacksonObjectMapper());
       //将自己的消息转化器加入容器中，一般自己的是排在最后的，一般情况下用不到，所以加一个0索引，就排在第一轮
       converters.add(0, converter);
   }
   ```

---

## 4. 查询时隐藏敏感字段（密码脱敏）

在查询操作时，为了不让前端看到用户的密码，可以在 Service 层处理：

```java
Employee employee = employeeMapper.getById(id);
// 密码脱敏
employee.setPassword("****************");
```

---

## 5. 公共字段自动填充（AOP + 注解 + 反射）

### 5.1 问题分析

#### 📌 核心问题

**代码冗余、不便于后期维护。**

#### 🔍 问题背景与现象

1. **公共字段普遍存在**：在业务表（如 employee、category 等）中，都存在 4 个公共字段：
   - `create_time`（创建时间）
   - `create_user`（创建人id）
   - `update_time`（修改时间）
   - `update_user`（修改人id）
2. **重复的赋值代码**：每次执行新增（insert）或修改（update）操作时，都需要在业务代码中重复编写设置这 4 个字段的逻辑。
   - 例如：`employee.setCreateTime(...)`、`employee.setCreateUser(...)` 等。
   - 例如：`category.setCreateTime(...)`、`category.setCreateUser(...)` 等。

#### ⚠️ 带来的痛点

- **代码臃肿**：每个 Service 方法的写操作里都要写一遍这 4 行代码，显得非常啰嗦。
- **维护成本极高**：如果后期需要修改公共字段的赋值逻辑（比如把 `BaseContext.getCurrentId()` 改成别的获取方式），或者新增了一个公共字段，需要把所有涉及写操作的 Service 方法全部改一遍，极易遗漏，导致数据不一致。

### 5.2 解决思路

#### 一、公共字段与操作类型的映射关系

| 字段名 | 含义 | 数据类型 | 需要执行的操作类型 |
|---|---|---|---|
| `create_time` | 创建时间 | datetime | **insert** |
| `create_user` | 创建人id | bigint | **insert** |
| `update_time` | 修改时间 | datetime | **insert、update** |
| `update_user` | 修改人id | bigint | **insert、update** |

（注：`create_time` 和 `create_user` 仅在插入时赋值；`update_time` 和 `update_user` 在插入和更新时都需要赋值）

#### 二、代码层面实现步骤

1. **自定义注解 `AutoFill`**：
   - 创建一个自定义注解，用于标识需要进行公共字段自动填充的 Mapper 方法。
2. **自定义切面类 `AutoFillAspect`**：
   - 定义切面类，使用 AOP 统一拦截加入了 `@AutoFill` 注解的方法。
   - 在切面逻辑中，通过**反射**机制，为实体对象中的公共字段动态赋值（例如 `createTime`、`updateTime` 等）。
3. **在 Mapper 的方法上加入 `AutoFill` 注解**：
   - 在 Mapper 层的 `insert` 或 `update` 方法上标记 `@AutoFill`，指明当前操作是 INSERT 还是 UPDATE 类型，从而触发切面的自动填充逻辑。

#### 三、核心技术点

- **枚举**（用于区分 INSERT 和 UPDATE 等操作类型）
- **注解**（自定义 `@AutoFill`）
- **AOP**（面向切面编程，实现统一拦截与增强）
- **反射**（运行时动态获取类属性并赋值）

> [!tip] 思路总结
> 这套思路的核心是为了解决上"代码冗余、维护难"的问题。它的本质思想是**"抽取公共逻辑，利用 AOP 和反射实现无侵入式的自动填充"**。开发者只需在方法上打一个注解标记，切面就会在方法执行前（或后）自动往实体类里塞入当前时间、当前用户 ID 等数据，从而让业务代码彻底告别 `setCreateTime()` 等重复代码。

---

## 6. 今日重点

> [!star] 重点回顾
>
> 1. **DTO 的核心价值**：安全隔离（防恶意注入）、结构解耦（前后端与数据库解耦）、数据裁剪与聚合、独立参数校验、避免过度获取。
> 2. **分层架构位置**：Controller（DTO + @Valid）→ Service（DTO 转 Entity）→ Mapper/DAO（只处理 Entity）；查询返回时 Entity 转 VO/Response DTO。
> 3. **时间格式处理**：`@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")` 或扩展 SpringMVC 消息转化器。
> 4. **密码脱敏**：查询操作在 Service 层 `employee.setPassword("****************")`。
> 5. **公共字段自动填充**：自定义 `@AutoFill` 注解 + `AutoFillAspect` 切面 + 反射，自动填充 `create_time`、`create_user`、`update_time`、`update_user`，消除代码冗余。
> 6. **RESTful 规范**：POST 添加、GET 查询、PUT 修改。

---

## 7. 面试问题

> [!question] 🎤 面试题
> **问题**：为什么接口入参要用 DTO 而不是数据库实体类 Entity？苍穹外卖里公共字段（create_time、create_user、update_time、update_user）又是如何自动填充的？

**参考答案**：

1. **为什么用 DTO 而非 Entity**：
   - **安全隔离**：直接用 Entity 接参，前端可以恶意注入 `id`、`role`、`balance` 等敏感字段，直接入库会被越权篡改；
   - **解耦**：数据库字段改名不影响前端接口契约；
   - **裁剪/聚合**：列表页只返回精简字段；一个请求可聚合多表数据；
   - **独立校验**：不同业务场景（增/改/查）可各自定义校验注解，避免冲突。
2. **公共字段自动填充**：
   - 自定义注解 `@AutoFill`（区分 INSERT / UPDATE）标注在 Mapper 的 insert / update 方法上；
   - `AutoFillAspect` 切面拦截该注解，在方法执行前通过**反射**给实体的 create_time、create_user、update_time、update_user 动态赋值（createUser / updateUser 从 `BaseContext` 取）；
   - 本质：**抽取公共逻辑 + AOP + 反射**，无侵入消除重复的 `setCreateTime()` 代码。
