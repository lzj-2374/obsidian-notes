
**自动配置**：

什么是自动配置：
springboot的自动配置就是当spring项目启动后，一些配置类、bean对象就自动存入到IOC容器中，不需要我们去手动声明，从而简化了开发，省去了繁琐的配置操作。

自动配置实现方案一：
我们知道，一般项目的启动类有`@SpringBootApplication`注解，他下面的`@ComponentScan`，会自动扫描启动类所在包及其子类的bean,但是我们在引入第三方工具包，比如引入了一个itheima-utils包，我们要使用引入包的TokenParser，一般情况下扫描不到，我们可以在启动类里添加一个
`@ComponentScan（"要扫描的包","原有的包"）`来指定扫描范围，加了注解后，默认`@SpringBootApplication`不再生效。
但是，如果我们要扫描的包有很多，`@ComponentScan（"要扫描的包1","要扫描的包2",...,"原有的包"）`他就会变得好==繁琐==，并且==性能也会变低==。

自动配置实现方案二：
针对于刚才的问题，我们可以使用@Import导入。@Import导入的类会被Spring加载到IOC容器中，导入的形式主要有以下几种：
1，导入  普通类（TokenParser是普通类）
直接在启动类加`@Import（TokenParser.class)`
2，导入  配置类（HeaderParser是配置类，有@Configuration  @Bean）
直接在启动类加`@Import（HeaderParser.class）`
3，导入  ImportSelector 接口实现类(MyImportSelector是ImportSelector的实现类)
直接在启动类加`@Import(MyImportSelector.class)`批量导入

`MyImportSelector`,`HeaderParser`等全是第三方提供的，对开发人员不友好，所以我们可以创建一个注解`EnableXxxx`，里面将Import这个注解封装，使用时，直接在启动类加`@EnableXxxx`就行