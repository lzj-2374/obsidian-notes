# day09 商家端订单管理模块

## 一、整体分层（请求怎么走的）

B 端前端点"接单"按钮后，数据流动：

```
浏览器
  │  PUT /api/order/confirm  {"id": 3, "status": 3}
  ▼
nginx（location /api/ → 转发到后端 8080 的 /admin/）
  ▼
admin/OrderController.confirm()      ← 接住请求：收数据 → 调service → 打包返回
  ▼
OrderService（接口，只声明）
  ▼
OrderServiceImpl                      ← 真正干活：改什么状态、写哪个字段
  ▼
OrderMapper.update()                  ← 一条 SQL
  ▼
MySQL orders 表
```

## 二、admin/OrderController.java（新建）

### 2.1 类上的注解

```java
@RestController("adminOrderController")
@RequestMapping("/admin/order")
@Api(tags = "订单管理接口")
public class OrderController {
```

- `@RestController` = `@Controller` + `@ResponseBody`：处理 HTTP 请求、返回 JSON。
- **`("adminOrderController")` 的 bean 名很重要**：工程里已有 C 端 `user/OrderController`（bean 名 `userOrderController`），两个类同名，不加名字区分会启动报错（bean 重名冲突）。
- `@RequestMapping("/admin/order")`：本类所有接口的前缀。必须挂 `/admin` 下，因为 nginx 把 `/api/...` 改成 `/admin/...` 转发，且走管理员 JWT 拦截器。

### 2.2 八个接口

```java
@GetMapping("/conditionSearch")
public Result<PageResult> conditionSearch(OrdersPageQueryDTO ordersPageQueryDTO)
```
- 订单搜索（列表页）。**没有 `@RequestBody`**：前端是 GET，参数在 URL 上（`?number=&phone=&status=&beginTime=&endTime=`），Spring 自动绑定进 DTO。
- 返回 `PageResult`：`total` 总条数（画分页）+ `list` 当前页数据。

```java
@GetMapping("/statistics")
public Result<OrderStatisticsVO> statistics()
```
- 状态统计：待接单 / 已接单 / 派送中 三个数量，对应页面顶部 3 张卡片。

```java
@GetMapping("/details/{id}")
public Result<OrderVO> details(@PathVariable("id") Long id)
```
- 订单详情。`{id}` 是路径变量（`/details/3` 的 3），`@PathVariable("id")` 取出。

```java
@PutMapping("/confirm")
public Result confirm(@RequestBody OrdersConfirmDTO ordersConfirmDTO)
```
- 接单。PUT + `@RequestBody`：前端发 JSON 请求体，自动转成 DTO。
- `rejection`（拒单）、`cancel`（取消）同套路，DTO 里多个"原因"字段写进库展示。

```java
@PutMapping("/delivery/{id}")   // 派送
@PutMapping("/complete/{id}")    // 完成
```
- 没有 body，id 直接写在 URL 里，只用 `@PathVariable`。

> 规律：**前端怎么发，后端就怎么接**。URL 带参数 → 自动绑定 / `@PathVariable`；body 是 JSON → `@RequestBody`。写接口前先 F12 看前端发的东西。

## 三、OrderServiceImpl.java（业务逻辑）

### 3.1 conditionSearch（订单搜索）

```java
public PageResult conditionSearch(OrdersPageQueryDTO ordersPageQueryDTO) {
    PageHelper.startPage(ordersPageQueryDTO.getPage(), ordersPageQueryDTO.getPageSize());
    Page<OrderVO> page = orderMapper.pageQuery(ordersPageQueryDTO);
    List<OrderVO> orderVOList = page.getResult();

    if (!CollectionUtils.isEmpty(orderVOList)) {
        orderVOList.forEach(orderVO -> {
            if (orderVO.getStatus().equals(Orders.TO_BE_CONFIRMED)
                    || orderVO.getStatus().equals(Orders.CONFIRMED)
                    || orderVO.getStatus().equals(Orders.DELIVERY_IN_PROGRESS)) {
                List<OrderDetail> orderDetailList = orderDetailMapper.getByOrderId(orderVO.getId());
                List<String> orderDishList = orderDetailList.stream()
                        .map(x -> x.getName() + "*" + x.getNumber() + ";")
                        .collect(Collectors.toList());
                orderVO.setOrderDishes(String.join("", orderDishList));
            }
        });
    }
    return new PageResult(page.getTotal(), orderVOList);
}
```

- `PageHelper.startPage(page, pageSize)`：告诉 PageHelper 下一条查询取第几页、每页几条。
- **为什么只给 待接单/已接单/派送中 三种状态查明细？** 业务需要：商家列表页看到"待接单"单要直接知道"这单是啥菜"（如 `可乐*3;米饭*1;`）来决定接不接；已取消/已完成是死数据，查明细纯属浪费 SQL。
- stream 拆解：每行明细 → `"可乐"*3;` → 收集成 List → `String.join("")` 拼成 `"可乐*3;米饭*1;"` 存进 `OrderVO.orderDishes`，前端表格直接显示。

### 3.2 statistics（状态统计）

```java
public OrderStatisticsVO statistics() {
    Integer toBeConfirmed = orderMapper.countStatus(Orders.TO_BE_CONFIRMED);
    Integer confirmed = orderMapper.countStatus(Orders.CONFIRMED);
    Integer deliveryInProgress = orderMapper.countStatus(Orders.DELIVERY_IN_PROGRESS);
    OrderStatisticsVO vo = new OrderStatisticsVO();
    vo.setToBeConfirmed(toBeConfirmed);
    vo.setConfirmed(confirmed);
    vo.setDeliveryInProgress(deliveryInProgress);
    return vo;
}
```

对应 SQL：`select count(id) from orders where status = 2/3/4`。

### 3.3 details（订单详情）

```java
public OrderVO details(Long id) {
    Orders orders = orderMapper.getById(id);                                  // 1. 查订单
    List<OrderDetail> list = orderDetailMapper.getByOrderId(orders.getId());  // 2. 查明细
    OrderVO orderVO = new OrderVO();
    BeanUtils.copyProperties(orders, orderVO);        // 3. 字段逐个抄过去
    orderVO.setOrderDetailList(list);                 // 4. 明细塞进 VO
    return orderVO;
}
```

`OrderVO extends Orders`（多 `orderDishes`、`orderDetailList` 两个字段），能装下"订单+明细"整体返回给前端详情弹窗。

### 3.4 confirm（接单）——最简单的

```java
public void confirm(OrdersConfirmDTO ordersConfirmDTO) {
    Orders orders = Orders.builder()
            .id(ordersConfirmDTO.getId())
            .status(Orders.CONFIRMED)
            .build();
    orderMapper.update(orders);
}
```

**关键技巧**：`update` 的 XML 是动态 SQL（`<set>` 里 `<if test="xxx != null">`，**哪个字段不为 null 才更新哪个**）。只设 `id` 和 `status`，生成 `update orders set status=3 where id=?`，金额、收货人全不动。所以"只改状态"的操作都能复用同一个 `update`。

### 3.5 rejection（拒单）——带校验

```java
public void rejection(OrdersRejectionDTO ordersRejectionDTO) {
    Orders ordersDB = orderMapper.getById(ordersRejectionDTO.getId()); // 先查现状

    if (ordersDB == null || !ordersDB.getStatus().equals(Orders.TO_BE_CONFIRMED)) {
        throw new OrderBusinessException(MessageConstant.ORDER_STATUS_ERROR);
    }

    Integer payStatus = ordersDB.getPayStatus();
    if (payStatus == Orders.PAID) {
        // 模拟支付：没有微信商户号，不做真实退款，直接打日志
        log.info("拒单申请退款（模拟）：订单号{}", ordersDB.getNumber());
    }

    Orders orders = new Orders();
    orders.setId(ordersDB.getId());
    orders.setStatus(Orders.CANCELLED);        // 拒单 = 已取消
    orders.setRejectionReason(ordersRejectionDTO.getRejectionReason()); // 记拒单原因
    orders.setPayStatus(Orders.REFUND);         // 支付状态改 2（退款）
    orders.setCancelTime(LocalDateTime.now());
    orderMapper.update(orders);
}
```

**为什么先查？** 业务规则："只有待接单的订单才能拒"。对派送中的单点拒单，状态流转就乱了，所以校验现状，不满足抛异常（全局异常处理器转成前端报错"订单状态错误"）。

**与官方参考答案的差别**：参考答案调 `weChatPayUtil.refund(...)` 真实退款，但没有商户号会抛异常，故改为打日志 + 直接置 `pay_status=2`，效果等价。

### 3.6 cancel（商家取消订单）

和拒单几乎一样，区别：**拒单要求订单必须是待接单**；取消不限制状态（已接单也能取消），所以没有那个 if 校验。

### 3.7 delivery / complete（派送、完成）

```java
public void delivery(Long id) {
    Orders ordersDB = orderMapper.getById(id);
    if (ordersDB == null || !ordersDB.getStatus().equals(Orders.CONFIRMED)) {
        throw new OrderBusinessException(MessageConstant.ORDER_STATUS_ERROR); // 只有已接单(3)才能派送
    }
    Orders orders = new Orders();
    orders.setId(ordersDB.getId());
    orders.setStatus(Orders.DELIVERY_IN_PROGRESS); // 4
    orderMapper.update(orders);
}

public void complete(Long id) {
    Orders ordersDB = orderMapper.getById(id);
    if (ordersDB == null || !ordersDB.getStatus().equals(Orders.DELIVERY_IN_PROGRESS)) {
        throw new OrderBusinessException(MessageConstant.ORDER_STATUS_ERROR); // 只有派送中(4)才能完成
    }
    Orders orders = new Orders();
    orders.setId(ordersDB.getId());
    orders.setStatus(Orders.COMPLETED);            // 5
    orders.setDeliveryTime(LocalDateTime.now());    // 送达时间（营业额统计按此算）
    orderMapper.update(orders);
}
```

### 3.8 getOverTimeOrder（超时未支付）

```java
public List<Orders> getOverTimeOrder() {
    LocalDateTime time = LocalDateTime.now().minusMinutes(15);
    return orderMapper.getByOrderStatusAndOrderTimeLT(Orders.PENDING_PAYMENT, time);
}
```

SQL：`where status = 1 and order_time < 15分钟前`。找"待付款且下单超 15 分钟"的单，给定时任务/WebSocket 调用（本模块未接定时任务）。

## 四、Mapper 层新增方法

### OrderMapper.java

```java
@Select("select count(id) from orders where status = #{status}")
Integer countStatus(Integer status);
```

### OrderDetailMapper.java

```java
@Select("select * from order_detail where order_id = #{orderId}")
List<OrderDetail> getByOrderId(Long orderId);
```

之前只有 `getByIds`（批量查多单），详情/搜索需要单个查，故补一个。

> SQL 短、表简单 → `@Select` 注解直接写；复杂动态 SQL 才放 XML。

## 五、状态流转图（8 个接口串起来）

```
        下单(1 待付款)
           │ 支付成功
           ▼
        2 待接单 ──confirm接单──► 3 已接单 ──delivery派送──► 4 派送中 ──complete完成──► 5 已完成
           │                        │
           │rejection拒单            │cancel取消
           ▼                        ▼
        6 已取消 ◄───────────────────┘
```

每个接口 = 在图上推一格：先查现状（`getById`）→ 校验当前格子对不对 → `update` 到新格子。推错方向（对"已完成"的单点接单）就抛 `ORDER_STATUS_ERROR`。

## 六、改了哪些文件

| 文件 | 改动 | 原因 |
|---|---|---|
| `admin/OrderController.java` | 新建，8 个接口 | B 端入口，之前完全缺失（404 根源） |
| `OrderService.java` | 加 7 个方法声明 | 接口层声明 |
| `OrderServiceImpl.java` | 实现 7 个 + 补全空壳 `conditionSearch` | 业务逻辑 |
| `OrderMapper.java` | 加 `countStatus` | 统计卡片 |
| `OrderDetailMapper.java` | 加 `getByOrderId` | 查明细 |

## 七、验证

重启 sky-server（bean 名、新 Controller 都是启动时注册的）→ B 端"订单管理"页刷新。
若某接口 404/405，把 F12 里该请求的 Method + URL 对照上表排查。
