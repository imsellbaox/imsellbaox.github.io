---
layout: post
title:  SpringBoot自动装配原理
categories: [编程开发,Java]
excerpt: 1.当SpringBoot应用程序启动时，先创建SpringApplication对象，在....
---
1.当SpringBoot应用程序启动时，先创建SpringApplication对象，在其构造方法中进行初始化工作：判断项目类型、初始化参数、监听器，加载Spring.factories文件。



2.执行run方法，先调用prepareContext初始化一些属性、环境对象、获取上下文配置等一些准备工作，再一个就是注册启动类为beanDenifition



3.然后调用refreshContext。这个方法会调用Spring的refresh方法，一共13个方法来完成整个Spring的启动，总结一下就是:基础配置、BeanFactory的创建处理、注解解析，bean的注册和实例化、监听器以及根据项目的一些扩展（例如Tocat容器的加载）。其中最重要的是invokeBeanFactoryPostProcessors，它是BeanFactoryPostProcessor和BeanDefinitionRegistry PostProcessor的子类。它将动态注册bean到容器中，解析@Configration的所有类的注解（@Componet、@ComponetScan、@import....），解析完成之后把这些bean注册到BeanFactory中。



其中@import非常有意思，他会从主类开始递归找出所有@import的类，然后进行分类，用不同的ImportSelector的子类处理解析@import的类。

@EnableAutoConfigration原理也是导入一个ImportSelector调用selectImport()方法，进一步调用getCandidateConfigurations方法扫描Spring.factories文件并且导出需要的自动配置类（之中传入该自动配置类的属性配置类），再通过反射机制实例化为对应的标注@Configuration形式的IoC容器配置类，然后注入IoC容器。实现自动装配。