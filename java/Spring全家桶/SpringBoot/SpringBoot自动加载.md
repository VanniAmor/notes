# SpringBoot自动加载原理剖析



以hello world的demo为例，探究SpringBoot的自动加载

```java
@SpringBootApplication
public class FirstApplication {

    public static void main(String[] args) {
        SpringApplication.run(FirstApplication.class);
    }
}
```



## @SpringBootApplication注解



@SpringBootApplication 这个注解表明当前类SpringBoot的主配置类，SpringBoot就运行这个类的main方法来启动SpringBoot应用



```java
## @SpringBootApplication注解源码

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(
    excludeFilters = {@Filter(
    type = FilterType.CUSTOM,
    classes = {TypeExcludeFilter.class}
), @Filter(
    type = FilterType.CUSTOM,
    classes = {AutoConfigurationExcludeFilter.class}
)}
)
public @interface SpringBootApplication {
```



在@SpringBootApplication里面包含有@SpringBootConfiguration和@EnableAutoConfiguration这两个重要的注解，下面进行分别解读



## @SpringBootConfiguration注解

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {
    @AliasFor(
        annotation = Configuration.class
    )
    boolean proxyBeanMethods() default true;
}
```

@SpringBootConfiguration： SpringBoot的配置类

​	标注在某个类上，表示这是一个SpringBoot的配置类



源码中又包含@Configuration这个注解

### @Configuration注解

这个是Spring中的常用注解，表明这是一个Spring程序的配置类

@Configuration中也包含了@Component注解，说明了配置类也是容器中的一个组件

​	

## @EnableAutoConfiguration注解 - 开启SpringBoot自动注册



@EnableAutoConfiguration，开启自动配置功能，这个注解告诉SpringBoot框架需要开启自动配置功能



```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {
```

源码中包含两个重要的注解 **@AutoConfigurationPackage 和 **@Import**

- @AutoConfigurationPackage	将类路径下的所有META-INF的所有Spring_factories中的自动加载类导入
- @Import  	将注解中声明的类导入到项目中，这里是导入 **AuthConfigurationImportSelector** 这个类



### @AutoConfigurationPackage - 注册基础扫描器

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import({Registrar.class})
public @interface AutoConfigurationPackage {
```

其中通过@Import注解，向容器中注册一个组件，这个Register是 **AutoConfigurationPackages**的一个内部类，如下

```java
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {
        Registrar() {
        }

        public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
            AutoConfigurationPackages.register(registry, (String[])(new AutoConfigurationPackages.PackageImports(metadata)).getPackageNames().toArray(new String[0]));
        }

        public Set<Object> determineImports(AnnotationMetadata metadata) {
            return Collections.singleton(new AutoConfigurationPackages.PackageImports(metadata));
        }
    }
```

其中的registerBeanDefinition方法，就是把主配置类（标注@SpringBootApplication的类）所在的包，注册为Spring的基础扫描器

这样Spring就自动扫描这个包下的所有组件



### @Import(AutoConfigurationImportSelector.class) - 全类名导入组件

![image-20200930143627397](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20200930143627397.png)

返回一个String数组，这个数组是SpringBoot下所有需要自动加载的组件的全类名

这些组件会被添加到Spring容器中

会给容器中导入非常多的自动配置类（xxxAutoConfiguration）；就是给容器中导入这个starter需要的所有组件，并配置好

![image-20200930144859471](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20200930144859471.png)

这些自动配置类，是各个厂商根据SpringBoot规范进行编写的，这些自动配置类免去了手动配置和注入的工作

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
        List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
        Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
        return configurations;
    }
```

![image-20200930150402405](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20200930150402405.png)

​	通过这段代码得知，项目最终是从每个包下面的**META-INF/spring.factories** 获取EnableConfiguration指定的值，并转化为Properties

​	![image-20200930150905228](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20200930150905228.png)

SpringBoot框架所有自动配置功能相关的包都在 **spring-boot-autoconfigure-2.3.4.RELASE.jar** 

![image-20200930151253429](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20200930151253429.png)