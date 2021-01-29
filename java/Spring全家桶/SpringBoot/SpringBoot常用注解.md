https://blog.csdn.net/JWbonze/article/details/108781475

### @SpringBootApplication

​	表明SpringBoot程序启动入口类，等同于以下几个注解之和：

- @SpringBootConfiguration：表示Application作为配置文件存在
- @EnableAutoConfiguration：表示启用SpringBoot内置的自动配置功能
- @ComponentScan： 扫描Bean，路径为Application类所在package以及package下的子路径，在SpringBoot中bean都放置在该路径以及子路径下。



## 表现层



### @Controller

![image-20201017114815175](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201017114815175.png)

Spring框架提供的



### @RestController

用于处理HTTP请求，@RestController = @Controller + @ResponseBody

### @ResponseBody

将Controller方法返回的对象通过适当的转换器转换为指定的格式后，写入到response对象的body区，通常用来返回JSON数据或XML数据

注意：在使用此注解后不会再走视图处理器，而是直接将数据写入到输出流，效果等同于通过response对象输出指定格式的数据

一般在异步获取数据时使用，如AJAX

### @RequestMapping

用于配置url映射

@GetMapping组合注解相当于@RequestMapping(method = RequestMethod.GET)

@PostMapping组合注解相当于@RequestMapping(method = RequestMethod.POST)

### @RequestParam

将请求参数绑定到控制器的方法上，是SpringMVC中接收普通参数的注解

![image-20201017121710544](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201017121710544.png)

### @Service

### @Repository

### @Component

### @Value @Autowired @Resource

### @PropertySource

指定读取的配置文件，格式如下

```java
// 指定从person.properties中读取配置
@PropertySource(value = {"classpath:person.properties"})
```

### @ImportResource 

导入Spring的配置文件，并让文件中的的内容生效

SpringBoot里面没有Spring的配置文件，我们自己编写的配置文件，SpringBoot不能自动识别

想让Spring配置文件生效，需要把 **@ImportResource标注在一个配置类上**,格式如下

```
// 加载指定的Spring配置文件
@ImportResource(locations = {"classpath:beans.xml"})
```

但是，SpringBoot推荐使用全注解的方式来给容器添加组件

具体是使用到 **@Bean和@Configuration** 这个注解

```java
// 指明当前类是一个配置类，用此来替代之前的Spring配置文件
@Configuration
public class MyAppConfig 
{
    // 将方法返回的值添加到容器中，容器中这个组件的ID就是方法名
    @Bean
    public HelloService helloService()
    {
        return new HelloService();
    }
}
```









