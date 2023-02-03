---
layout: post
title:  SpringBoot基础核心
categories: [编程开发,Java]
excerpt: SpringBoot内嵌Tomcat、jetty等容器，无需war包形式....
---
## SpringBoot基础功能：
### 1.独立运行Spring项目

SpringBoot可以通过jar包方式独立运行，打包为xxx.jar ，在cmd直接运行：java -jar xxx.jar

### 2.内嵌servlet容器

SpringBoot内嵌Tomcat、jetty等容器，无需war包形式部署项目。

### 3.提供sarter简化Maven

Spring提供一系列的starter，在pom中来简化Maven的依赖加载。

### 4.自动装配

SpringBoot 提供大量的自动装配类，会根据项目需求在类路径中的jar包、类自动装配Bean，减少配置。当然也提供自定义自动配置，让项目可以定制化。

### 5.准生产的应用监控

SpringBoot提供对http ssh telnet运行时的项目监控。

### 6.无代码生成与无代码xml配置

SpringBoot不是借助代码生成实现，使用大量注解以及条件注解生成对象、配置类。这是spring4.x提供的新特性

## SpringBoot优缺点：
### 优：
1.快速构建项目

2.对主流开发框架的自动配置

3.项目可独立运行，内嵌servlet容器

4.运行时的应用监控

5.极大提高开发效率

6.与云计算的天然集成

### 缺：
1.Spring体系及结构越来越大，需要研究才能上手

2. 版本迭代速度很快，需要不断学习

SpringBoot几个常用的注解

（1）@RestController和@Controller指定一个类，作为控制器的注解 

（2）@RequestMapping方法级别的映射注解

（3）@EnableAutoConfiguration和@SpringBootApplication是类级别的注解，用于启动类。自动完成Spring配置

（4）@Configuration类级别的注解   标注为一个配置类

（5）@ComponentScan类级别的注解，自动扫描加载所有的Spring组件包括Bean注入。启动类

（6）@ImportResource类级别注解，当我们必须使用一个xml的配置时，使用@ImportResource和@Configuration来导入。

（7）@Autowired注解，一般结合@ComponentScan注解，来自动注入一个Service或Dao级别的Bean

（8）@Component类级别注解，用来标识一个组件。

（9）@transactional注解，在方法就是方法事务，类上就是类事务.
