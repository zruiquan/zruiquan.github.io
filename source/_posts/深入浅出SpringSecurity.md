---
title: 深入浅出SpringSecurity
date: 2022-11-10 10:22:35
tags:
- SpringSecurity
categories:
- SpringSecurity
---

# 第 1 章  

Spring Security 虽然历史悠久，但是从来没有像今天这样受到开发者这么多的关注。究其原因，还是沾了微服务的光。作为 Spring 家族中的一员，在和 Spring 家族中的其他产品如SpringBoot、Spring  Cloud 等进行整合时，Spring  Security 拥有众多同类型框架无可比拟的优势。本章我们就先从整体上了解一下 Spring Security 及其工作原理。
本章涉及的主要知识点有：
●     Spring Security 简介。
●     Spring Security 整体架构。

## 1.1 Spring Security 简介  

Java 企业级开发生态丰富，无论你想做哪方面的功能，都有众多的框架和工具可供选择，以至于 SUN 公司在早些年不得不制定了很多规范，这些规范在今天依然影响着我们的开发，安全领域也是如此。然而，不同于其他领域，在 Java 企业级开发中，安全管理方面的框架非常少，一般来说，主要有三种方案：
●     Shiro
●     Spring Security
●     开发者自己实现

shiro 本身是一个老牌的安全管理框架，有着众多的优点，例如轻量、简单、易于集成、可以在 JavaSE 环境中使用等。不过， 在微服务时代， Shiro 就显得力不从心了，在微服务面前， 它无法充分展示自己的优势。

也有开发者选择自己实现安全管理，据笔者所知，这一部分人不在少数。但是一个系统的安全，不仅仅是登录和权限控制这么简单，我们还要考虑各种各样可能存在的网络攻击以及防御策略，从这个角度来说，开发者自己实现安全管理也并非是一件容易的事情，只有大公司才有足够的人力物力去支持这件事情。  

Spring Security 作为 Spring 家族的一员，在和 Spring 家族的其他成员如 Spring Boot、SpringCloud 等进行整合时，具有其他框架无可比拟的优势，同时对 OAuth2 有着良好的支持，再加上 Spring Cloud 对 Spring Security 的不断加持（如推出 Spring Cloud Security）， 让 Spring Security不知不觉中成为微服务项目的首选安全管理方案。  

