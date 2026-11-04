# MyBatis-Plus 零基础入门（知识点 + 代码 + 面试背诵）

> MyBatis-Plus（简称 MP）：**MyBatis 的增强工具，只增强不修改**，兼容原生 MyBatis；单表 CRUD 不用写 XML，大幅提升开发效率。

## 一、核心概念

1. **定位**：不是新 ORM 框架，底层仍然是 MyBatis，原有 MyBatis 代码完全保留，可以混合使用。
2. **核心组件**
    - `BaseMapper<T>`：Mapper 接口继承，内置单表 CRUD 方法（insert、deleteById、selectById、updateById、selectList 等）
    - `ServiceImpl`：Service 层通用实现类；`IService`：Service 接口，封装批量、分页等上层方法
    - **Wrapper 条件构造器**：构建动态查询 / 更新条件
        - `QueryWrapper`：查询条件
        - `UpdateWrapper`：更新条件
        - `LambdaQueryWrapper` / `LambdaUpdateWrapper`：Lambda 写法，**编译期校验字段名，防止字符串写错**（推荐）Baomidou™
3. **常用注解（实体类）**
    - `@TableName("user")`：指定数据库表名，实体类名和表名不一致时使用
    - `@TableId`：标记主键字段
        - `type = IdType.AUTO`：数据库自增
        - `type = IdType.ASSIGN_ID`：雪花算法（默认，分布式 ID）
    - `@TableField("db_column")`：普通字段映射，实体属性名和数据库列名不一致；`fill = FieldFill.INSERT` 自动填充
    - `@TableLogic`：逻辑删除标记
    - `@Version`：乐观锁版本号
4. **插件体系（3.5 + 推荐使用 Interceptor 注册）**
    - 分页插件：**物理分页**（不是查询全部内存分页，底层拦截 SQL 拼接 limit）
    - 逻辑删除插件、乐观锁插件、性能分析插件

## 二、SpringBoot 整合步骤（最简入门）

### 1）引入依赖

SpringBoot2

```
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.17</version>
</dependency>
```

SpringBoot3

```
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
    <version>3.5.17</version>
</dependency>
```

同时引入 mysql 驱动、lombok

### 2）application.yml 配置

```
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/mp_db?useUnicode=true&characterEncoding=utf8
    username: root
    password: your-password

mybatis-plus:
  configuration:
    map-underscore-to-camel-case: true # 开启下划线转驼峰（默认开启）
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl # 打印SQL日志
  global-config:
    db-config:
      id-type: assign_id # 全局主键策略
```

### 3）启动类添加注解

```
@SpringBootApplication
@MapperScan("com.xxx.mapper") //扫描mapper包
public class MpApplication {
    public static void main(String[] args) {
        SpringApplication.run(MpApplication.class, args);
    }
}
```

### 4）编写实体类 User.java

```
@Data
@TableName("user")
public class User {
    @TableId(type = IdType.AUTO)
    private Long id;
    private String name;
    private Integer age;
    private String email;
}
```

### 5）Mapper 接口，继承 BaseMapper

```
public interface UserMapper extends BaseMapper<User> {
    // 无需写任何方法，直接拥有CRUD
}
```

### 6）测试 CRUD

```
@Autowired
private UserMapper userMapper;

// 新增
userMapper.insert(new User(null,"张三",20,"xxx@qq.com"));
// 根据id查询
User user = userMapper.selectById(1L);
// 根据id更新
userMapper.updateById(user);
// 根据id删除
userMapper.deleteById(1L);
// 条件查询
List<User> list = userMapper.selectList(
    new LambdaQueryWrapper<User>()
        .eq(User::getName,"张三")
        .ge(User::getAge,18)
);
```

## 三、Wrapper 条件构造器常用方法

- `eq()` 等于；`ne()` 不等于
- `gt()`大于；`ge()`大于等于；`lt()`小于；`le()`小于等于
- `like()`模糊查询；`in()`范围；`orderByDesc()`排序
- `.and()` `.or()` 多条件逻辑拼接

> 特点：如果传入的值为 null，**默认不拼接该条件**，天然支持动态 SQL

## 四、高频进阶功能

1. **分页插件（必须注册插件，否则是逻辑分页）**

```
@Configuration
public class MpConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor(){
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}
```

分页使用：

```
Page<User> page = new Page<>(1,5); // 当前页，每页条数
Page<User> result = userMapper.selectPage(page, null);
List<User> records = result.getRecords(); // 当前页数据
long total = result.getTotal(); //总条数
```

2. **自动填充（创建时间 / 更新时间）** 实体字段：

```
@TableField(fill = FieldFill.INSERT)
private LocalDateTime createTime;
@TableField(fill = FieldFill.INSERT_UPDATE)
private LocalDateTime updateTime;
```

编写填充处理器：`implements MetaObjectHandler`，重写 insertFill、updateFill

3. **逻辑删除** 数据库增加字段 `deleted` 0 未删，1 已删 实体标记：`@TableLogic`

> deleteById 不会真实删除，执行 update set deleted=1

4. **乐观锁**：版本号`@Version`，更新时对比版本号，适用于并发更新场景
5. **代码生成器 AutoGenerator**：一键生成 Entity、Mapper、Service、Controller

## 五、面试题【题目 + 详细解答 + 快速背诵】

### 题目 1：MyBatis 和 MyBatis-Plus 关系？

**详细解答** MyBatis-Plus 是 MyBatis 的增强工具，**只做增强不做修改**，不是替代 MyBatis。底层依旧使用 MyBatis，原有 MyBatis XML、自定义 Mapper 方法完全兼容。MP 封装通用单表 CRUD、条件构造器、分页、逻辑删除等能力，减少重复代码；复杂多表关联查询仍然可以手写 MyBatis 原生 SQL。

**快速背诵** MP 是 MyBatis 增强工具，只增强不修改，完全兼容原生 MyBatis；简化单表 CRUD，复杂 SQL 依旧手写。

### 题目 2：BaseMapper 和 IService/ServiceImpl 区别？

**详细解答** BaseMapper：Mapper 层接口，提供基础 CRUD（insert、selectById），数据库直接操作。 IService + ServiceImpl：Service 层封装，在 BaseMapper 之上封装批量操作、分页、链式查询等上层能力，推荐业务层调用 Service。

**快速背诵** BaseMapper：Mapper 基础单表操作；ServiceImpl 在它之上封装批量、分页等高层 API。

### 题目 3：QueryWrapper 和 LambdaQueryWrapper 区别？

**详细解答** QueryWrapper：传入字符串字段名，运行期才发现字段写错； LambdaQueryWrapper：使用实体类方法引用`User::getName`，**编译期校验字段**，开发推荐，减少 bug。

**快速背诵** QueryWrapper 传字符串；LambdaQueryWrapper 用方法引用，编译检查，推荐优先使用。

### 题目 4：MP 分页插件原理？

**详细解答** 基于 MyBatis 拦截器`Interceptor`，SQL 执行前拦截原始 SQL，自动拼接`limit`语句实现**物理分页**，查询只返回当前页数据，不会一次性加载全部数据到内存。必须手动注入 PaginationInnerInterceptor 插件，否则`Page`对象不会拼接 limit，变成内存分页。

**快速背诵** 拦截器拦截 SQL，自动拼接 limit 物理分页；必须配置分页插件，否则为内存分页。

### 题目 5：MP 有哪些常见坑？

1. Wrapper 查询时，字段 null 会自动忽略，容易出现条件丢失；
2. 分页**必须配置插件**，否则假分页；
3. 批量插入不要循环调用 mapper.insert，性能差，使用 service 批量方法；
4. 多表复杂关联查询不建议强行用 Wrapper，直接写 XML 更易维护；
5. 主键策略不要混用，分布式场景优先 ASSIGN_ID 雪花 ID。