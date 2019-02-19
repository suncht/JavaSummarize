# Spring Boot启动过程

## 一、前言
JavaWeb项目开发中大多数都会使用到Spring框架，随着项目越来越庞大，Spring的XML文件配置越来多，使用起来越来越繁琐，此时SpringBoot给我们新理念“约定大于配置”，使用Properties属性文件替代繁琐的XML配置，成为了Java微服务界的标配脚手架，SpringCloud就是基于SpringBoot快速搭建。所以我们需要了解SpringBoot启动原理

## 二、容器启动
spring boot一般是指定容器启动main方法，然后以命令行方式启动Jar包
```java
@SpringBootApplication
public class DubboClientApplication {
	public static void main(String[] args) throws Exception {
		SpringApplication.run(DubboClientApplication.class, args);
	}
}
```
核心关注2个东西：
1. @SpringBootApplication注解
2. SpringApplication.run()静态方法

### 2.1 @SpringBootApplication注解
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
}
```
核心注解：
> @SpringBootConfiguration（实际就是个@Configuration）：表示这是一个JavaConfig配置类，可以在这个类中自定义bean,依赖关系等。-》这个是spring-boot特有的注解，常用到。

> @EnableAutoConfiguration：借助@Import的帮助，将所有符合自动配置条件的bean定义加载到IoC容器（建议放在根包路径下，这样可以扫描子包和类）。-》这个需要详细深挖！

> @ComponentScan：spring的自动扫描注解，可定义扫描范围，加载到IOC容器。-》这个不多说，spring的注解大家肯定眼熟

其中@EnableAutoConfiguration这个注解的源码：
```java
@SuppressWarnings("deprecation")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
```