## 一. SpringBoot如何在Tomcat容器下启动的原理
启动原理前提：
> 1. 根据Servlet3.0规范，服务器启动（web应用启动）会创建当前web应用里面每一个jar包里面ServletContainerInitializer实例。
> 2. ServletContainerInitializer的实现放在jar包的META-INF/services文件夹下，有一个名为javax.servlet.ServletContainerInitializer的文件，内容就是ServletContainerInitializer的实现类的全类名。
> 3. 使用@HandlesTypes，在应用启动的时候加载我们感兴趣的类。

1. SpringBoot 中的 ServletContainerInitializer 实现类位置在spring-web模块下：META-INF/services/javax.servlet.ServletContainerInitializer文件， 内容是：org.springframework.web.SpringServletContainerInitializer
2. SpringServletContainerInitializer查找WebApplicationInitializer接口实现，执行onStartup方法
SpringBootServletInitializer就是WebApplicationInitializer的一种核心实现，onStartup方法中调用SpringApplication.run方法

**总结核心点：**
1. **ServletContainerInitializer**就是利用JAVA SPI技术，Servlet3.0规范中规定`META-INF/services/javax.servlet.ServletContainerInitializer`文件
2. **@HandlesTypes**定义要加载指定接口的类，SpringBoot中是WebApplicationInitializer接口
3. 集成SpringBootServletInitializer

