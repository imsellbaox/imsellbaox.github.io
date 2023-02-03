---
layout: post
title:  SpringBoot总汇
categories: [编程开发,Java]
excerpt: 所有技术框架都遵循一条主线规律：从一个复杂的应用场景衍生出一种规范....
---
<font color=red>所有技术框架都遵循一条主线规律：从一个复杂的应用场景衍生出一种规范框架，人们只需要进行各种配置而不去手动实现它，就能完成一些强大的功能；发展到一定程度后，人们根据所需要的生产实际情况，选取其中的一些功能和设计精华，重构出一些轻量级框架；之后为了提高开发效率，把各种配置趋向于自动化，提倡出“约定大于配置”，进而衍生出一站式的解决方案。</font>

![](..\images\Springboot\image1.png)

## 1.SpringBoot是什么？
Spring Boot 基于 Spring 开发，它致力于帮助Spring基础框架开发更加快速、方便、敏捷的plus框架！Spring Boot 以约定大于配置的核心思想，帮我们自动配置大量默认设置，同时它还提供大量第三方工具库的配置（例如 Redis、MongoDB、JPA、RabbitMQ、Quartz 等等）

## 2.微服务架构

微服务架构是一种架构概念，旨在通过功能的分解构建一个个微小的离散服务进行解耦，供更加灵活的服务支持。

概念： 把一个大型的单个应用程序和服务拆分为数个甚至数十个的支持微服务，它可扩展单个组件。

定义：以一个个业务领域组件来创建应用程序，每个组件都可以独立地进行开发，管理，迭代。在组件中使用云架构和平台式部署、管理和服务功能，使产品交付变得更加简单。

关注两个问题：

<font color=red>横向扩展解决:</font> 一台服务器用户满了，再加一台服务器

<font color=red>负载均衡解决：</font>A服务器占用98%资源，B服务器占用10%

### 传统开发模式

![](..\images\Springboot\image2.png)

### 微服务架构

![](..\images\Springboot\image3.png)

### SpringBoot源码分析

[SpringBoot自动装配原理](https://blog.csdn.net/VB551/article/details/121299648?spm=1001.2014.3001.5501)

### SpringBoot基础核心

[SpringBoot基础核心](https://blog.csdn.net/VB551/article/details/121347549?spm=1001.2014.3001.5501)