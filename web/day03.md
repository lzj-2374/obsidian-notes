# Day 03 笔记

---

## 一、分页查询 — 原始方法

### 基本概念

| 概念 | 说明 |
|------|------|
| 每页展示数 | 前端写好即可 |
| 总记录数 | 需要后端返回给前端 |
| 总页数 | 前端自行计算：`总条数 ÷ 每页展示数` |

### 核心 SQL

**分页查询（LIMIT）**

```sql
-- 格式：limit 起始索引, 每页展示记录数（起始索引从 0 开始）
SELECT e.*, d.name FROM emp e LEFT JOIN dept d ON e.dept_id = d.id LIMIT 0, 5;
```

**数据总数**

```sql
-- 用聚合函数 count(*) 查询总条数
SELECT COUNT(*) FROM emp e LEFT JOIN dept d ON e.dept_id = d.id;
```

> **速记：** `LIMIT` 起始索引 = `(page - 1) × pageSize`。

### 前后端交互

**前端传递给后端：**
- `page` — 当前页码
- `pageSize` — 每页展示记录数

**后端返回给前端：**
- `List<T> rows` — 当前页数据列表
- `Long total` — 总记录数

一般封装到实体类中：

```java
@Data
public class PageResult<T> {
    private List<T> rows;
    private Long total;
}
```

> **速记：** 分页返回两样：一页数据 + 总条数，封装到 PageResult 里。

### 三层架构分工

| 层级 | 职责 |
|------|------|
| **Controller** | 接收分页参数 `page`、`pageSize`；调用 Service；返回 `PageResult` |
| **Service** | 调用 Mapper 查总记录数；调用 Mapper 查当前页数据；封装 `PageResult` |
| **Mapper** | 执行两条 SQL：`SELECT COUNT(*)` 查总数；`SELECT ... LIMIT` 查当前页 |

请求示例：`/emps?page=1&pageSize=10`

> **速记：** Controller 收参调服务，Service 组装结果，Mapper 干 SQL 活。

---

## 二、分页查询 — PageHelper 插件

PageHelper 是第三方 MyBatis 分页插件，**简化分页操作，提高开发效率**。

### 使用步骤

**1. 引入依赖**

```xml
<!-- 分页插件 PageHelper -->
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
    <version>1.4.7</version>
</dependency>
```

**2. Mapper 接口方法无需改动**，正常写不分页的 SQL 即可。

**3. Service 层开启分页**

```java
// 设置分页参数：第 start 页，每页 pageSize 条
PageHelper.startPage(start, pageSize);
```

### 注意事项

> **速记：** 分两件事记住——SQL 末尾别加分号，PageHelper 只对紧挨着的第一条查询生效。

1. **SQL 语句结尾不能加分号** `;`，否则报错
2. **PageHelper 仅对紧跟其后的第一条查询语句生效**，后续查询不受影响

---

## 三、日期时间参数接收

前端传递日期格式，后端需用 `@DateTimeFormat` 注解指定格式进行转换：

```java
@DateTimeFormat(pattern = "yyyy-MM-dd")
LocalDate begin;
```

> **速记：** 日期进 Java 要注解，`@DateTimeFormat` 指定 `yyyy-MM-dd` 格式。

---

## 四、MyBatis 多条件分页查询注意事项

### 1. 模糊查询 — `LIKE` 写法

❌ 错误：`#{}` 不能放在引号内

```sql
-- 错误写法，会报语法错
LIKE '%#{name}%'
```

✅ 正确：用 `CONCAT` 函数拼接

```sql
LIKE CONCAT('%', #{name}, '%')
```

> **速记：** LIKE 模糊查，`#{}` 不能放引号里，用 CONCAT 来拼。

### 2. `#{}` 外部不要手动加单引号

MyBatis 会自动处理字符串和日期的引号，手动加会导致解析异常。

> **速记：** `#{}` 自带引号，外面别再套单引号。

### 3. `null` 值查询大坑

MySQL 中 `字段 = NULL` 永远查不到数据（NULL 不能用 `=` 比较），不能直接写死条件。

> **速记：** MySQL 里 NULL 不用等号判，要用 IS NULL。

### 4. 动态 SQL — `<where>` + `<if>`

多条件筛选必须搭配动态 SQL：

```xml
<where>
    <if test="name != null and name != ''">
        AND name LIKE CONCAT('%', #{name}, '%')
    </if>
    <if test="gender != null">
        AND gender = #{gender}
    </if>
    <if test="begin != null">
        AND entry_date &gt;= #{begin}
    </if>
    <if test="end != null">
        AND entry_date &lt;= #{end}
    </if>
</where>
```

- **`<where>`**：自动处理多余的 `AND` 关键字
- **`<if>`**：参数为 null 或空字符串时，不拼接该条件段

> **速记：** `<where>` 自动去多余 AND，`<if>` 判空过滤无效条件。

**判断规则：**
- **字符串**：同时判 `!= null` 和 `!= ''`
- **数字 / 日期**：只判 `!= null`

---

## 五、请求参数封装 — 查询参数对象

### 问题

Controller 方法**参数太多**，新增筛选条件时方法签名越来越长，难维护。

### 解决方案：封装到实体类

定义 `EmpQueryParam` 查询参数类：

```java
@Data
public class EmpQueryParam {
    private Integer page = 1;          // 默认第 1 页
    private Integer pageSize = 10;     // 默认每页 10 条
    private String name;               // 员工姓名（模糊查询）
    private Integer gender;            // 性别
    @DateTimeFormat(pattern = "yyyy-MM-dd")
    private LocalDate begin;           // 入职开始日期
    @DateTimeFormat(pattern = "yyyy-MM-dd")
    private LocalDate end;             // 入职结束日期
}
```

### Controller 写法

```java
@GetMapping
public Result page(EmpQueryParam empQueryParam) {
    log.info("分页查询，参数：{}", empQueryParam);
    PageResult<Emp> pageResult = empService.page(empQueryParam);
    return Result.success(pageResult);
}
```

SpringMVC 自动把 URL 查询参数绑定到对象的属性上。

请求示例：`/emps?name=张&gender=1&begin=2007-09-01&end=2022-09-01&page=1&pageSize=10`

> **速记：** 参数太多就封装，一个对象全搞定，新增条件不改动方法签名。

### 优点

1. 参数统一管理，新增条件只需在类里加属性
2. 代码简洁，可读性强
3. 可以直接给参数设置**默认值**（如 `page=1`，`pageSize=10`）

### 配套三层改动

| 层级 | 改动内容 |
|------|---------|
| **Controller** | 方法入参改为 `EmpQueryParam` 对象 |
| **Service** | `page` 方法入参改为 `EmpQueryParam` |
| **Mapper** | 传入 `EmpQueryParam` 对象，XML 中用 `#{属性名}` 取值 |

---

## 六、MyBatis `<foreach>` 批量插入

### 作用

遍历集合，动态拼接 SQL，**实现批量插入多条记录**（一对多场景，如一次性插入员工多条工作经历）。

### foreach 五个核心属性

| 属性 | 作用 | 示例值 |
|------|------|--------|
| `collection` | 传入的集合/数组参数名 | `exprList` |
| `item` | 遍历时的元素别名 | `expr` |
| `separator` | 每条记录之间的分隔符 | `,` |
| `open` | 循环开始前拼接的片段 | `(`（可省略） |
| `close` | 循环结束后拼接的片段 | `)`（可省略） |

### XML 写法

```xml
<insert id="insertBatch">
    INSERT INTO emp_expr (emp_id, begin, end, company, job) VALUES
    <foreach collection="exprList" item="expr" separator=",">
        (#{expr.empId}, #{expr.begin}, #{expr.end}, #{expr.company}, #{expr.job})
    </foreach>
</insert>
```

### Mapper 接口

```java
void insertBatch(@Param("exprList") List<EmpExpr> exprList);
```

> ✅ **重点：** 如果参数是单独集合而非实体属性，**必须加 `@Param` 给集合起名**，和 `collection` 的值对应！

### 业务场景（员工 + 工作经历）

员工和工作经历是**一对多**关系：1 个员工 → 多条工作经历记录

**修改员工时的处理逻辑：**
1. 先删除该员工旧的全部工作经历
2. 使用 `<foreach>` 批量插入前端提交的新工作经历列表

> **速记：** `foreach` 遍历集合拼 SQL，批量插入一对多；集合参数必加 `@Param`。

---

## 七、MySQL 事务

### 概念

事务是一组操作的**不可分割的集合**，所有操作要么**同时成功，要么同时失败**。

> ⚠️ MySQL 默认事务是**自动提交**的，执行每条 DML 语句时会立即隐式提交。

### 操作步骤

```sql
-- 1. 开启事务
START TRANSACTION;
-- 或
BEGIN;

-- 2. 执行多条 SQL
... sql ...

-- 3. 全部成功 → 提交；有失败 → 回滚
COMMIT;   -- 提交事务
ROLLBACK; -- 回滚事务
```

### 使用场景

- 银行转账（扣款 + 存款，必须同时成功）
- 下单扣减库存（创建订单 + 减少库存，必须一致）

> **速记：** 事务保证原子性，要么全成要么全败，转账下单最常见。

---

## 📌 今日重点

| # | 知识点 | 一句话记住 |
|---|--------|-----------|
| 1 | 手动分页公式 | `LIMIT (page-1)*pageSize, pageSize` |
| 2 | PageHelper 使用 | `startPage()` 后紧跟第一条查询才生效，SQL 末尾别加分号 |
| 3 | `LIKE` 模糊查询 | `CONCAT('%', #{name}, '%')`，`#{}` 不放引号内 |
| 4 | 动态 SQL | `<where>` 去多余 AND，`<if>` 判空过滤条件 |
| 5 | 参数封装对象 | 参数多就封装，新增条件不改方法签名 |
| 6 | `<foreach>` 批量插入 | 五个属性背下来，集合参数必加 `@Param` |
| 7 | 事务 | 一组操作原子性，要么全成功要么全回滚 |
