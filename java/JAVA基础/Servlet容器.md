Servlet本质，其实就是一个Java程序



## Web容器

![img](https://img-blog.csdn.net/20140119014801968?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc25hcmxmdXR1cmU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

Web服务器使用HTTP协议来传输数据。最简单的一种情况是，用户在浏览器（客户端，client）中输入一个URL（如，www.programcreek.com/static.html），然后就能获取网页进行阅览。因此，服务器完成的工作就是发送网页至客户端。传输过程遵循HTTP协议，它指明了请求（request）消息和响应（response）消息的格式。

但此时用户/客户端只能向服务器请求静态网页。如果想服务器根据用户输入动态响应数据，需要引入CGI等机制

![img](https://img-blog.csdn.net/20170309183226078?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTGl1Tmlhbl9TaVl1/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

对Java而言，Servlet就是其CGI程序

## Servlet容器 & Servlet

Servlet容器的基本思想是在服务端使用Java来动态生成网页， Servlet容器是一个装载一堆Servlet对象的容器，并具有管理Servlet对象的功能

![img](https://img-blog.csdnimg.cn/20190519224628114.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2ZnODgxMjE4,size_16,color_FFFFFF,t_70)

**Servlet 是 javax.servlet 包中定义的接口**，其声明了Servlet周期的三个基本方法：init()、service() 和 destroy()。

- **init()** 方法在 Servlet 生命周期的初始化阶段调用。它被传递一个实现 javax.servlet.ServletConfig 接口的对象，该接口允许 Servlet 从 Web 应用程序访问初始化参数。
- **service()** 方法在初始化后对每个请求进行调用。每个请求都在自己的独立线程中提供服务。Web容器为每个请求调用 Servlet 的 service() 方法。service() 方法确认请求的类型，并将其分派给适当的方法来处理该请求。
- **destroy()** 方法在销毁 Servlet 对象时调用，用来释放所持有的资源。

从Servlet对象的生命周期中，我们可以看到Servlet类是由类加载器动态加载到容器的。每个Http请求的处理都在独自的线程中，Servlet对象可以同时服务多个线程（线程不安全），当它不再使用时，会被JVM进行GC



Servlet在JVM中运行，Servlet容器负责Servlet的创建、执行和销毁。

Servlet容器的主要功能是将请求转发到正确的Servlet进行处理，并在JVM处理完后将动态生成的结果返回到正确的位置。Servlet允许JVM在处理每个请求时使用单独的java线程，这是Servlet容器的一个主要优点。



## Tomcat & Apache & Nginx

在B/S架构中，客户端发起http请求，请求经过服务器，服务器调用CGI程序来动态处理请求，并生成结果，最后服务器把CGI结果处理为符合http协议的格式返回给B端

其中，Apache 和 Nginx都是扮演的服务器的角色，而Tomcat是CGI程序，是Servlet容器（当然，Tomcat也是一个轻量级的Web服务器，可以处理静态内容，但是性能并不优越）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190407101654161.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDIyMTYxMw==,size_16,color_FFFFFF,t_70)



Apache全称是 Apache Http Server Project，是同步多进程模型，一个连接对应一个进程

Nginx是异步进程模型，使用多路IO复用技术，多个连接（万级别）可以对应一个进程