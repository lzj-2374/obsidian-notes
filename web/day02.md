# Day 02 笔记

---

## 一、删除部门请求流程

浏览器 → Controller（接收参数 `/depts?id=1`，调用 Service） → Service（调用 Mapper） → Mapper（执行 `delete from dept where id=?`） → 返回统一响应 `Result`

---

## 二、Controller 层接收请求参数的三种方式

### 方式一：通过 HttpServletRequest 手动获取（繁琐，不推荐）

```java
@DeleteMapping()
public Result delete(HttpServletRequest request) {
    String idStr = request.getParameter("id");   // 获取到的 id 为 String 类型
    int id = Integer.parseInt(idStr);            // 手动类型转换
    log.info("根据id删部门：" + id);
    return Result.success();
}
```

> **速记：** 原始 API 手动解析，步骤多易出错，少用。

### 方式二：使用 `@RequestParam` 注解绑定参数

```java
@DeleteMapping("/depts")
public Result delete(@RequestParam("id", required = true) Integer deptId) {
    log.info("根据id删部门：" + deptId);
    return Result.success();
}
```

> **速记：** `@RequestParam` 显式绑定，`required=true`（默认）不传就报错，可改 `false` 放宽限制。

### 方式三：形参名与请求参数名相同，直接定义形参（推荐 ⭐）

```java
@DeleteMapping("/depts")
public Result delete(Integer id) {
    log.info("根据id删部门：" + id);
    return Result.success();
}
```

> **速记：** 名字一样直接接，省注解最省事，Spring 自动绑定。

---

## 三、Mapper 层 — 删除 SQL

```java
// 使用 #{} 占位符，Spring 自动生成预编译 SQL（防 SQL 注入）
@Delete("delete from dept where id=#{id}")
```

> **速记：** `#{}` 是预编译占位符，防注入；`${}` 直接拼串，危险不用。

---

## 四、新增部门 — 接收 JSON 参数

前端 POST 请求，请求体传递 JSON：

```
POST /depts
{"name": "教研部"}
```

使用实体对象接收，**必须加 `@RequestBody` 注解**：

```java
@PostMapping("/depts")
public Result add(@RequestBody Dept dept) {
    log.info("新增部门：" + dept);
    return Result.success();
}
```

> **速记：** JSON 放请求体用 `@RequestBody`，键名要和属性名一致才能自动匹配。

**适用场景：** `POST`、`PUT` 请求中通过请求体传递参数。

---

## 五、修改部门 — 查询回显（路径变量）

```java
@GetMapping("/depts/{id}")
public Result getInfo(@PathVariable Integer id) {
    log.info("根据id查询部门：" + id);
    return Result.success();
}
```

多个路径变量：

```java
@GetMapping("/depts/{id}/{name}")
public Result getInfo(@PathVariable Integer id, @PathVariable String name) {
    // ...
}
```

> **速记：** URL 路径里的动态段用 `{}` 占位，`@PathVariable` 取值，适合 GET 查询。

**请求路径拼接规则：** 类上 `@RequestMapping` 的 value + 方法上 `@GetMapping` 的 value，公共前缀（如 `/depts`）建议提到类级别。

---

## 六、日志技术

### 为什么不用 `System.out.println`？

- 只能输出到控制台
- 不方便扩展和维护

日志的作用：**数据追踪、性能优化、问题排查、系统监控**

### 引入与使用

Spring Boot 已自动引入 Logback，无需手动添加依赖。

```java
// 定义日志记录器
private static final Logger log = LoggerFactory.getLogger(LogTest.class);
```

### 日志级别（由低到高）

| 日志级别 | 说明 | 记录方式 |
|---------|------|---------|
| `trace` | 追踪程序运行轨迹，项目中很少使用 | `log.trace("...")` |
| `debug` | 调试信息，开发中常用最低级别 | `log.debug("...")` |
| `info`  | 普通运行信息，记录关键事件（IO、连接等） | `log.info("...")` |
| `warn`  | 警告信息，潜在风险，程序仍可运行 | `log.warn("...")` |
| `error` | 错误信息，发生异常 | `log.error("...")` |

### 核心知识点

1. **级别越高，信息越严重**；设置日志输出级别后，**只会打印 ≥ 该级别的日志**。
   > 例：配置级别为 `info` → 只输出 info、warn、error；trace、debug 被过滤。

2. 开发环境常用 `debug`，生产环境一般设为 `info` 或 `warn`，减少日志量。

3. Spring Boot 默认日志级别：**`info`**

4. 在配置文件中灵活控制输出类型（**大于等于配置级别的日志才会输出**）

### 关闭/开启日志（logback.xml）

```xml
<!-- 开启：debug；关闭：off；全开：all -->
<root level="debug">
    <appender-ref ref="STDOUT" />
</root>
```

> **速记：** 设低级别打得多，设高级别打得少，`off` 直接关闭。

---

## 七、多表关系与多表查询

### 一对多 / 多对一

实现方式：在"**多**"的一方表中添加字段，关联"**一**"的一方的主键。

### 笛卡尔积

两张表所有行两两配对，结果集庞大且大部分无效。**多表联查必须加关联条件过滤无效数据。**

> **速记：** 两表相乘出笛卡尔积，不加条件全是废数据。

---

### 内连接（INNER JOIN）

查询两张表的**交集**数据。

**隐式内连接：**
```sql
SELECT 字段列表 FROM 表1, 表2 WHERE 连接条件;
```

**显式内连接（推荐）：**
```sql
SELECT 字段列表 FROM 表1 [AS] 别名1 INNER JOIN 表2 [AS] 别名2 ON 连接条件;
```

> `INNER` 和 `AS` 均可省略。

> **速记：** 内连接取交集，ON 写条件 AS 可省。

---

### 外连接

**左外连接（LEFT JOIN）：** 左表全部保留，右表无匹配填 NULL

**右外连接（RIGHT JOIN）：** 右表全部保留，左表无匹配填 NULL

`OUTER` 关键字可省略。

```sql
-- 左外连接
SELECT 字段 FROM 表1 LEFT JOIN 表2 ON 关联条件;  -- 表1 全部保留

-- 右外连接
SELECT 字段 FROM 表1 RIGHT JOIN 表2 ON 关联条件; -- 表2 全部保留
```

> **速记：** 左连保左，右连保右，外字可省互换等价。

**关键结论：** 左右外连接互换表位置结果相同。
> `A LEFT JOIN B` ≡ `B RIGHT JOIN A`

---

### 子查询（嵌套查询）

在 SQL 语句中嵌套 SELECT 语句。

**基础形式：**
```sql
SELECT * FROM t1 WHERE column1 = (SELECT column1 FROM t2 ...);
```

**分类：**

| 类型 | 说明 |
|------|------|
| 标量子查询 | 子查询返回**单个值** |
| 列子查询 | 子查询返回**一列数据** |
| 行子查询 | 子查询返回**一行数据** |
| 表子查询 | 子查询返回**多行多列**，相当于临时表 |

> **速记：** 套娃 SELECT 叫子查询，返回单值/单列/单行/多行多列分四类。
