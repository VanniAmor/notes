![image-20201017110406083](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201017110406083.png)

![image-20201017110434902](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201017110434902.png)



## 注解定义



https://www.runoob.com/w3cnote/java-annotation.html

https://blog.csdn.net/zt15732625878/article/details/100061528

![image-20201017112853334](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201017112853334.png)



![image-20201017113113179](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201017113113179.png)

![image-20201017113504145](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201017113504145.png)

```java

public class AnnotationTest {
    public static void main(String[] args) {
        try {
            //获取Person的Class对象
            Person person = Person.builder().build();
            Class clazz = person.getClass();
            //判断person对象上是否有Info注解
            if (clazz.isAnnotationPresent(Info.class)) {
                System.out.println("Person类上配置了Info注解！");
                //获取该对象上Info类型的注解
                Info infoAnno = (Info) clazz.getAnnotation(Info.class);
                System.out.println("person.name :" + infoAnno.value() + ",person.isDelete:" + infoAnno.isDelete());
            } else {
                System.out.println("Person类上没有配置Info注解！");
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```



![image-20201017114328893](https://gitee.com/Vanni/pic-bed/raw/master/img/image-20201017114328893.png)