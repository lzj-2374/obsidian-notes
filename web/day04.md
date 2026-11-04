# Day 04 笔记

---

## 一、MySQL 事务（复习）

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

## 二、Spring 事务管理 — `@Transactional`

### 注解说明

`@Transactional` 将当前方法交给 Spring 进行事务管理：
- 方法开启前 → **开启事务**
- 成功执行完毕 → **提交事务**
- 出现异常 → **回滚事务**

**推荐位置：** Service 层方法上，也可放在类上或接口上。

### 调试日志配置

在 `application.yml` 中配置，可打印 Spring 事务管理的底层日志，方便调试：

```yaml
logging:
  level:
    org.springframework.jdbc.support.JdbcTransactionManager: debug
```

> **速记：** 事务底层日志 debug 开，调试注解看细节，生产环境别开着。

---

## 三、事务进阶 — `rollbackFor`

`@Transactional` 默认只在**运行时异常**（RuntimeException）时回滚；如果手动抛出普通 Exception，**不会回滚**。

解决办法：指定 `rollbackFor` 包含所有异常：

```java
// 所有异常都会触发回滚
@Transactional(rollbackFor = {Exception.class})
```

> **速记：** 默认只回滚运行时异常，想全回滚加 `rollbackFor = Exception.class`。

---

## 四、事务进阶 — 传播行为（Propagation）

### 概念

当一个加了事务的方法，被另一个带事务的方法调用时，**被调用方法怎么处理事务**。

### 六种传播属性

| 传播行为 | 说明 |
|---------|------|
| `REQUIRED`（默认） | 有事务就加入，没有就新建 |
| `REQUIRES_NEW` | **一定创建全新独立事务**，不管有没有已有事务 |
| `SUPPORTS` | 有事务就加入，没有就无事务执行 |
| `NOT_SUPPORTED` | 挂起已有事务，以无事务方式运行 |
| `MANDATORY` | 强制必须有事务，没有则抛异常 |
| `NEVER` | 禁止事务，存在则抛异常 |

### 典型场景：独立记录操作日志

**问题：** 在 `EmpServiceImpl.update()` 中用 `try...finally` 记录操作日志，但未单独控制事务，一旦前面出错，日志也会一起回滚，导致失败记录丢失。

```java
@Transactional(rollbackFor = {Exception.class})
@Override
public void update(Emp emp) {
    try {
        emp.setUpdateTime(LocalDateTime.now());
        empMapper.update(emp);
        // ... 其他操作
    } finally {
        // 记录操作日志
        EmpLog empLog = new EmpLog(null, LocalDateTime.now(), "操作信息：" + emp);
        empLogService.insertLog(empLog);
    }
}
```

**解决：** 在 `EmpLogServiceImpl.insertLog()` 上加 `REQUIRES_NEW`，创建独立事务，即使外层回滚，日志也能保存。

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void insertLog(EmpLog empLog) {
    empLogMapper.insert(empLog);
}
```

> **速记：** 日志要独立存，`REQUIRES_NEW` 开新事务，外层回滚它不受影响。

---

## 五、事务四大特性（ACID）

| 特性 | 说明 |
|------|------|
| **原子性（Atomicity）** | 事务是不可分割的最小单元，要么全部成功，要么全部失败 |
| **一致性（Consistency）** | 事务执行后，数据库数据保持合法一致的状态 |
| **隔离性（Isolation）** | 多个并发事务之间互相隔离，互不干扰 |
| **持久性（Durability）** | 事务提交/回滚后，修改永久保存，断电不丢失 |

> **速记：** ACID 四特性，原子一致隔离持久，背下来。

---

## 六、文件上传

### 前端表单三要素

```html
<form action="/upload" method="post" enctype="multipart/form-data">
    姓名: <input type="text" name="name"><br>
    年龄: <input type="text" name="age"><br>
    图像: <input type="file" name="file"><br>
    <input type="submit" value="上传文件">
</form>
```

| 要素 | 要求 | 原因 |
|------|------|------|
| 请求方式 | `method="post"` | GET 无法传输大量文件数据 |
| 编码类型 | `enctype="multipart/form-data"` | 把文件转为二进制传输，默认编码不能传文件 |
| 表单元素 | `<input type="file" name="file">` | `name` 不能省略 |

> **速记：** 上传表单三件套：POST、multipart编码、type=file。

### 后端接收

Spring Boot 使用 `MultipartFile` 接收，**参数名要和 input 的 `name` 保持一致**：

```java
@Slf4j
@RestController
public class UploadController {

    @PostMapping("/upload")
    public Result upload(String name, Integer age, MultipartFile file) throws IOException {
        log.info("接收参数：{}, {}, {}", name, age, file);
        return Result.success();
    }
}
```

### 保存到本地

```java
// 获取原始文件名
String originalFilename = file.getOriginalFilename();
// 保存到本地
file.transferTo(new File("D:/桌面/javaweb/ziliao/" + originalFilename));
```

> **速记：** `getOriginalFilename()` 拿原名，`transferTo()` 存本地。

### 解决文件名冲突 — 使用 UUID

第二次上传同名文件会覆盖第一次，用 UUID 生成唯一文件名：

```java
String originalFilename = file.getOriginalFilename();
// 截取后缀名
String extension = originalFilename.substring(originalFilename.lastIndexOf("."));
// 生成唯一文件名
String newFileName = UUID.randomUUID().toString() + extension;
// 保存
file.transferTo(new File("D:/桌面/javaweb/ziliao/" + newFileName));
```

> **速记：** 同名怕覆盖，UUID 拼接后缀名，保证唯一不冲突。

### 修改上传大小限制

Spring Boot 默认只能上传 **≤ 1MB** 的文件，在 `application.yml` 中修改：

```yaml
spring:
  servlet:
    multipart:
      # 最大单个文件大小
      max-file-size: 10MB
      # 最大请求大小（包含所有文件和表单数据）
      max-request-size: 100MB
```

> **速记：** 默认 1MB 太小，yml 里改 `max-file-size` 和 `max-request-size`。

---

## 七、阿里云 OSS 对象存储

### 为什么用云服务？

本地/服务器存储的问题：磁盘满了扩容麻烦，服务器坏了数据丢失。云存储更安全、可靠、低成本。

### 什么是 OSS？

**OSS（Object Storage Service）** 是阿里云提供的海量、安全、低成本、高可靠的云存储服务，支持存储文本、图片、音频、视频等各种文件。

### 核心概念

- **Bucket（存储空间）：** 存放文件的容器，所有文件（Object）必须归属某一个 Bucket
- **SDK（Software Development Kit）：** 软件开发工具包，包含依赖 jar 包、代码示例等

### 使用步骤

注册实名 → 充值 → 开通 OSS → 创建 Bucket → 获取 AccessKey 密钥 → SDK 编码 → 项目集成

> **速记：** 云存储避坑，OSS 注册创建桶，AK 密钥配 SDK。

---

## 八、删除员工 — 一对多关联查询封装

删除单条数据也是一种特殊的批量删除，只需开发一个接口。

### 结果映射要点

**1. 一对多 — `<collection>` 标签**

把多条子表数据封装到主实体的 `List` 集合属性：

- `property`：主实体中**集合属性名**
- `ofType`：集合里**单个元素的实体类型**（不是 List）

```xml
<collection property="exprList" ofType="EmpExpr">
    ...
</collection>
```

**2. 一对一 — `<association>` 标签**

封装单个关联对象：

- `property`：主实体中对象属性名
- `javaType`：关联对象的实体类型

```xml
<association property="dept" javaType="Dept">
    ...
</association>
```

**3. `resultMap` vs `resultType`**

- `resultType`：只能用于简单单表查询
- **一对多 / 一对一关联查询必须使用 `resultMap`**

**4. 多表联查注意事项**

- 同名字段要设置别名，消除字段歧义
- `LEFT JOIN` 一对多查询会产生多条主表重复记录，MyBatis 依靠主表 ID 自动合并，将子表数据存入集合
- XML 中不能用 Java 的 `+` 拼接字符串，`@Select` 注解可以

**5. SQL 易错点**

字符串拼接时关键字前后缺少空格，会造成 SQL 单词粘连，触发语法报错。

> **速记：** 一对多用 collection，一对一 association，关联查询 resultMap 不能忘。

---

## 九、MyBatis 动态 SQL 知识点

### 核心概念

根据传入参数的**不同条件**，动态拼接 SQL 语句，解决多条件查询时手动拼接容易多出 `AND`/`OR` 的问题。

### 常用标签

#### 1. `<if>` — 条件判断

满足条件才拼接里面的 SQL，`test` 里直接写属性名，不用 `#{}`：

```xml
<if test="name != null and name != ''">
    AND name LIKE CONCAT('%', #{name}, '%')
</if>
```

> **速记：** `if` 判空拼接，字符串双判，数字日期判 null。

#### 2. `<where>` — 自动处理多余 AND/OR

自动去掉第一个多余的 `AND`/`OR`，但**只会去掉开头的，末尾处理不了**。

```xml
<where>
    <if test="name != null and name != ''">
        AND name LIKE CONCAT('%', #{name}, '%')
    </if>
    <if test="gender != null">
        AND gender = #{gender}
    </if>
</where>
```

> **速记：** `where` 自动去开头多余 AND，末尾还需自己注意。

#### 3. `<set>` — 更新操作自动去多余逗号

```xml
<set>
    <if test="name != null">name = #{name},</if>
    <if test="phone != null">phone = #{phone},</if>
</set>
```

> **速记：** `set` 自动去末尾多余逗号，update 专属。

#### 4. `<foreach>` — 遍历集合（批量删除/插入）

```xml
<!-- 批量删除 -->
DELETE FROM emp WHERE id IN
<foreach collection="ids" item="id" open="(" separator="," close=")">
    #{id}
</foreach>
```

五个核心属性：`collection`、`item`、`separator`、`open`、`close`

> **速记：** `foreach` 遍历拼 IN 语句，批量操作靠它。

#### 5. `<choose>` / `<when>` / `<otherwise>` — 多选一

类似 Java 的 `switch-case`，**只会执行第一个满足条件的 `<when>`**，`<otherwise>` 是默认分支。

```xml
<choose>
    <when test="name != null">AND name = #{name}</when>
    <when test="gender != null">AND gender = #{gender}</when>
    <otherwise>AND 1=1</otherwise>
</choose>
```

> **速记：** `choose` 多选一，`when` 匹配走起，`otherwise` 兜底。

---

## 十、全局异常处理器 — `GlobalExceptionHandler`

在 Controller 层定义统一异常处理类：

### 基础写法

```java
@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler
    public Result handleException(Exception e) {
        log.info("程序出错了~", e);
        return Result.error("出错了");
    }
}
```

> 只会提示"程序出错了"，不知道具体错误原因。

### 细化异常处理

针对特定异常定义处理方法，**优先级从下往上**，匹配到优先返回：

```java
@ExceptionHandler
public Result handleDuplicateKeyException(DuplicateKeyException e) {
    log.info("程序出错了！", e);
    String message = e.getMessage();
    int i = message.indexOf("Duplicate entry");
    String errMsg = message.substring(i);
    String[] arr = errMsg.split(" ");
    return Result.error(arr[2] + " 已存在");
}
```

> 提示信息：`xxx 已存在`

> **速记：** `@RestControllerAdvice` 全局抓异常，细分异常写方法，优先级从下往上找。

---

## 📌 今日重点

| # | 知识点 | 一句话记住 |
|---|--------|-----------|
| 1 | `@Transactional` 默认回滚 | 只回滚运行时异常，要全回滚加 `rollbackFor = Exception.class` |
| 2 | 传播行为 `REQUIRES_NEW` | 创建独立事务，外层回滚它不受影响，适合记日志 |
| 3 | 文件上传三要素 | POST + multipart/form-data + type=file |
| 4 | UUID 防覆盖 | `randomUUID()` + 截取后缀名，唯一不冲突 |
| 5 | 上传大小限制 | 默认 1MB，yml 改 `max-file-size` |
| 6 | `<collection>` vs `<association>` | 一对多用 collection，一对一用 association |
| 7 | 动态 SQL 标签 | if/where/set/foreach/choose 五个标签记住 |
| 8 | 全局异常处理 | `@RestControllerAdvice` + `@ExceptionHandler`，细分异常从下往上匹配 |
