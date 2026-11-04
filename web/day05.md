# Day 05 笔记

---

## 一、MySQL 统计查询 — CASE 函数

### 场景：统计每种职位对应的人数

**方式一（表达式形式）：**

```sql
SELECT
    (CASE job
        WHEN 1 THEN '班主任'
        WHEN 2 THEN '讲师'
        WHEN 3 THEN '学工主管'
        WHEN 4 THEN '教研主管'
        WHEN 5 THEN '咨询师'
        ELSE '其他'
    END) AS pos,
    COUNT(*) AS num
FROM emp
GROUP BY job
ORDER BY num;
```

**方式二（条件形式，推荐 ⭐）：**

```sql
SELECT
    (CASE
        WHEN job = 1 THEN '班主任'
        WHEN job = 2 THEN '讲师'
        WHEN job = 3 THEN '学工主管'
        WHEN job = 4 THEN '教研主管'
        WHEN job = 5 THEN '咨询师'
        ELSE '其他'
    END) AS pos,
    COUNT(*) AS num
FROM emp
GROUP BY job
ORDER BY num;
```

> **速记：** `CASE WHEN` 做枚举转中文名，`ELSE` 兜底，推荐条件写法更清晰。

### 统计男女员工数量 — IF 函数

```sql
-- IF(条件, true值, false值)
SELECT
    IF(gender = 1, '男员工', '女员工') AS name,
    COUNT(*) AS value
FROM emp
GROUP BY gender;
```

> **速记：** `IF(条件,真值,假值)` 三目运算符，分组统计一行搞定。

---

## 二、为什么统计查询用 `List<Map<String, Object>>`？

### 问题：不用实体类，为什么用 Map？

这条 SQL 是**临时统计查询**，结果只有两列：`pos`（职位名称字符串）、`num`（人数数字）。

**用 Map 的原因：**

1. 没有专门的实体类 POJO，没必要为一次统计新建 Java 类
2. Map 的 **key = 列名**，**value = 列值**，天然匹配查询结果
3. 一行数据 = 一个 Map，多行 = List，最终 `List<Map<String, Object>>`

> 如果写实体类，得新建 `JobCount` 类，写两个属性 + get/set，麻烦。**一次性统计查询，优先 Map。**

### 为什么 value 写 `Object`，不写 `String` 或 `Integer`？

因为一行 Map 里有**两种不同类型的数据**：
- `pos` → 字符串 `String`
- `num` → 数字 `Integer`

Java 中一个 Map 的 value 只能定义**同一种类型**：
- 写 `Map<String, String>`：数字被强转成字符串，前端图表不好用
- 写 `Map<String, Integer>`：pos 字符串放不进去，直接报错

👉 **`Object` 是所有类的父类，可以同时接收 String、Integer 任意类型。**

### Mapper 方法

```java
List<Map<String, Object>> countEmpJobData();
```

| 外层 | 内层 |
|------|------|
| `List`：多条统计记录（多个职位） | `Map`：单条记录，key=字段名，value=字段值 |

### 完整返回数据样例

```json
[
  {"pos": "班主任", "num": 5},
  {"pos": "讲师", "num": 12},
  {"pos": "咨询师", "num": 8}
]
```

前端拿到直接渲染图表（条形图），完美适配。

> **速记：** 临时统计用 Map，value 写 Object 兼容多类型，一行一个 Map，多行套 List。

---

## 三、`@Options` 注解 — 获取自增主键回填

### 作用

搭配 `@Insert` 使用，**拿到数据库自动生成的自增主键 id，回填到传入的实体类对象里**。

```java
@Options(useGeneratedKeys = true, keyProperty = "id")
@Insert("insert into emp (...) values (...)")
void insert(Emp emp);
```

### 参数解释

| 参数 | 说明 |
|------|------|
| `useGeneratedKeys = true` | 开启获取数据库自动生成的主键（MySQL 自增主键才生效） |
| `keyProperty = "id"` | 把拿到的主键值，赋值给**实体类的 id 属性**（注意：写的是属性名，不是数据库字段名） |

### 完整流程

```
1. 传入 emp 对象，此时 emp.getId() → null
2. 执行 insert 插入数据库，数据库自动生成 id=10
3. @Options 生效，把 10 回填给 emp 对象的 id 属性
4. 插入完成后，emp.getId() 就能拿到 10
```

### 使用场景

新增后**立刻要拿到主键 id**，例如：
- 新增员工，同时批量插入员工的工作经历（需要员工 id 作为外键）

```java
Emp emp = new Emp();
emp.setName("周瑜");
empMapper.insert(emp);
// 插入完成后，emp 对象里面自动带上数据库生成的 id
System.out.println(emp.getId()); // 输出：10
```

### 注意事项

> ⚠️ Oracle 没有自增主键，此注解不适用，仅 MySQL 自增主键场景。

> **速记：** `@Options` 插入后拿自增 id，`useGeneratedKeys=true` + `keyProperty` 指定属性名。

---

## 四、班级学员管理 — 下拉框查询员工列表

### 问题：新增班级时，班主任下拉框没有数据

**原因：** 需要从员工管理接口查出所有员工，供下拉框展示。

### 接口定义

```java
/**
 * 新增班级下拉框 — 查询员工列表
 */
@GetMapping("/list")
public Result find() {
    log.info("下拉框查询员工列表");
    List<Emp> empList = empService.find();
    return Result.success(empList);
}
```

---

### 难点解析

#### 一、为什么路径是 `/list`？

因为 `EmpController` 上已有 `@RequestMapping("/emps")`，所以完整路径是 `GET /emps/list`。

**`/list` 不能省略**，因为 Controller 中同时存在：

```java
@GetMapping("/list")   // 字面量路径，优先级高
@GetMapping("/{id}")   // 路径变量，优先级低
```

Spring 匹配规则：**字面量优先于路径变量**

```
请求 /emps/list
  ├─ /list  → 精确匹配 ✅ 走这里
  └─ /{id}  → 把 "list" 当 id，转 Integer 报错 ❌
```

如果不写 `/list`，请求就会被 `/{id}` 截胡，报：
> `Failed to convert value of type 'java.lang.String' to required type 'java.lang.Integer'`

这也解释了为什么前端下拉框一开始是"无数据"。

> **速记：** 路径变量 `/{id}` 和字面量 `/list` 并存，字面量优先，不写就截胡报错。

#### 二、为什么方法没有参数？

前端调用时什么都没传：

```javascript
mn.get("/emps/list")  // 裸请求，无参数
```

下拉框要的就是**全部员工**，不需要筛选条件，所以无需参数。

> ⚠️ **`@RequestBody` 不能出现在这里！**

```java
public Result find(@RequestBody Emp emp)  // ❌ 错误！
```

| 原因 | 说明 |
|------|------|
| GET 没有请求体 | `@RequestBody` 读请求体，GET 请求没有体，读不到 |
| 前端没传数据 | 传了也是 null |
| 业务不需要 | 下拉框查全部，无筛选条件 |

判断方法：只在前端确实要传筛选条件时才加参数（如按姓名查员工），否则不写。

> **速记：** GET 请求没请求体，`@RequestBody` 只能用在 POST/PUT 上。

#### 三、为什么返回 `List<Emp>` 而不是单个 `Emp`？

看前端怎么用这份数据：

```html
<el-option v-for="W in empList" :label="W.name" :value="W.id" />
```

`v-for` 遍历的是**数组**，所以后端必须返回列表。

| 接口 | 返回类型 | 场景 |
|------|---------|------|
| `GET /emps/list` | `List<Emp>` | 下拉框，要一堆候选 |
| `GET /emps/{id}` | `Emp` | 编辑回显，只要这一个 |

> **速记：** 下拉框要列表，编辑回显要单个，看前端怎么遍历。

#### 四、完整调用链（四层打通）

```
EmpController  @GetMapping("/list")
       ↓
EmpService     List<Emp> find();
       ↓
EmpServiceImpl  return empMapper.list(null, null, null, null);
       ↓
EmpMapper       List<Emp> list(...);  ← 复用现有方法，不用改
```

最后一层是复用的，`list(name, gender, begin, end)` 本身是带条件查询，把条件全传 `null` 就等于**无条件全查**。

#### 五、四个 `null` 为什么能查全表？

`EmpMapper.xml` 中的动态 SQL：

```xml
<where>
    <if test="name != null and name != ''">e.name LIKE CONCAT('%', #{name}, '%')</if>
    <if test="gender != null">AND e.gender = #{gender}</if>
    <if test="begin != null and end != null">AND e.entry_date BETWEEN #{begin} AND #{end}</if>
</where>
```

三个 `<if>` 条件全不成立 → 都不拼接 → `<where>` 标签内为空 → **连 WHERE 关键字都不输出**。

实际执行的就是：

```sql
SELECT e.*, d.name AS deptName
FROM emp e LEFT JOIN dept d ON e.dept_id = d.id
ORDER BY e.update_time DESC
```

> **速记：** `<if>` 全为 null 就不拼接，`<where>` 不输出，等同于无条件查询。

#### 六、完整数据链路

```
① 前端发起    mn.get("/emps/list")
     ↓
② 后端查询    empMapper.list(null, null, null, null) → 查全表
     ↓
③ 返回数据    [{id:5, name:"岳不群", ...}, {id:7, name:"令狐冲", ...}]
     ↓
④ 前端渲染    <el-option :label="W.name" :value="W.id" />
              → 用户看到"岳不群"，选中后存 masterId = 5
     ↓
⑤ 保存班级    把 masterId 插进 clazz.master_id
     ↓
⑥ 列表显示    JOIN emp 把 id 翻译回姓名
```

#### 一句话总结

> **下拉框接口 = 无参 GET 请求 + 返回列表 + 复用带条件的查询方法（条件全传 null）**
> 路径特意写成 `/list`，是为了躲开 `/{id}` 的路径变量匹配。

---

## 📌 今日重点

| # | 知识点 | 一句话记住 |
|---|--------|-----------|
| 1 | CASE 函数统计 | `CASE WHEN` 枚举转中文名，条件写法更清晰，推荐 |
| 2 | IF 函数统计 | `IF(条件,真值,假值)` 两分类统计一行搞定 |
| 3 | 统计查询用 Map | 临时统计无实体类，`List<Map<String,Object>>` 最省事 |
| 4 | value 为什么是 Object | 一行里有 String 也有 Integer，Object 统一兼容 |
| 5 | `@Options` 拿自增 id | `useGeneratedKeys=true` + `keyProperty` 指定属性名 |
| 6 | 下拉框路径 `/list` | 字面量优先于 `/{id}`，不写就被截胡报类型转换错误 |
| 7 | GET 无 `@RequestBody` | GET 没有请求体，下拉框查全部不需要参数 |
| 8 | 四个 null 查全表 | `<if>` 全为 null 不拼接，`<where>` 不输出关键字 |
