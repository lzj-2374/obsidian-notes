bean作用域

如果我们声明一个bean，但是没有给他指定名称，默认名字是类名（首字母要小写）

 `@RestController`（组合注解） 没有直接封装其他类，但 Spring MVC 在处理它时，底层会依赖几个非常核心的 Bean 类：

- **`RequestMappingHandlerMapping`**：启动时，这个 Bean 会扫描所有带有 `@Controller` 或 `@RestController` 的 Bean，解析里面的 `@RequestMapping`、`@GetMapping` 等注解，建立 **URL 到方法的映射关系**。当一个 HTTP 请求进来时，就是它负责找到对应的 Controller Bean 和方法。
    
- **`RequestMappingHandlerAdapter`**：找到对应的方法后，由这个 Bean 负责真正去执行该方法，并处理参数绑定（比如把 JSON 转为 DTO、把路径变量转为方法参数）。
    
- **`HttpMessageConverter`**（消息转换器）：因为 `@RestController` 包含了 `@ResponseBody`，请求返回时，`MappingJackson2HttpMessageConverter` 等转换器会将返回的 Java 对象（如 `Result`、`PageResult`）序列化为 JSON 格式写回给前端。