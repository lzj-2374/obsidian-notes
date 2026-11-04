# 数据封装

## 核心规则

实体类属性名和数据库查询返回的字段名**一致**，MyBatis 会自动封装；否则不会自动封装。

```java
// 数据库字段名
id, username, create_time, update_time
```

```java
// 实体类属性名
private Integer id;          // ✅ 能自动封装
private String name;         // ✅ 能自动封装
private LocalDateTime createTime;  // ❌ 不能（下划线 vs 驼峰）
private LocalDateTime updateTime;   // ❌ 不能（下划线 vs 驼峰）
```

> **速记：** 名字一样自动封，不一样就 null。

---

## 解决方法

### 方法一：手动结果映射（@Results）

在 Mapper 层的方法上添加注解，手动建立映射关系：

```java
@Results({
    @Result(column = "create_time", property = "createTime"),
    @Result(column = "update_time", property = "updateTime")
})
```

> **速记：** `@Results` 手动配对，`column` 对库字段，`property` 对属性。

### 方法二：SQL 起别名

在写 SQL 语句时，直接为字段起一个与实体属性一致的别名：

```java
@Select("select id, name, create_time createTime, update_time updateTime from dept order by update_time desc")
```

> **速记：** SQL 里起别名，让库字段长得和属性一样。

### 方法三：开启驼峰命名映射（推荐 ⭐）

在 `application.yml` 中开启全局驼峰命名自动转换，一劳永逸：

```yaml
# 输入关键字：camel
mybatis:
  configuration:
    map-underscore-to-camel-case: true
```

> **速记：** 一行配置全搞定，下划线自动变驼峰，推荐首选！
