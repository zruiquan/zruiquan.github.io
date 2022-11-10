---
title: 深入浅出SpringSecurity
date: 2022-11-10 10:22:35
tags:
- SpringSecurity
categories:
- SpringSecurity
---

# 第 1 章

## 1.1 Spring Security 简介  

Java 企业级开发生态丰富，无论你想做哪方面的功能，都有众多的框架和工具可供选择，以至于 SUN 公司在早些年不得不制定了很多规范，这些规范在今天依然影响着我们的开发，安全领域也是如此。然而，不同于其他领域，在 Java 企业级开发中，安全管理方面的框架非常少，一般来说，主要有三种方案：
●     Shiro
●     Spring Security
●     开发者自己实现

shiro 本身是一个老牌的安全管理框架，有着众多的优点，例如轻量、简单、易于集成、可以在 JavaSE 环境中使用等。不过， 在微服务时代， Shiro 就显得力不从心了，在微服务面前， 它无法充分展示自己的优势。

也有开发者选择自己实现安全管理，据笔者所知，这一部分人不在少数。但是一个系统的安全，不仅仅是登录和权限控制这么简单，我们还要考虑各种各样可能存在的网络攻击以及防御策略，从这个角度来说，开发者自己实现安全管理也并非是一件容易的事情，只有大公司才有足够的人力物力去支持这件事情。  

Spring Security 作为 Spring 家族的一员，在和 Spring 家族的其他成员如 Spring Boot、SpringCloud 等进行整合时，具有其他框架无可比拟的优势，同时对 OAuth2 有着良好的支持，再加上 Spring Cloud 对 Spring Security 的不断加持（如推出 Spring Cloud Security）， 让 Spring Security不知不觉中成为微服务项目的首选安全管理方案。  

陈年旧事  

Spring Security 最早叫 Acegi Security， 这个名称并不是说它和 Spring 就没有关系，它依然是为 Spring 框架提供安全支持的。 Acegi Security 基于 Spring，可以帮助我们为项目建立丰富的角色与权限管理系统。 Acegi Security 虽然好用，但是最为人诟病的则是它臃肿烦琐的配置，这一问题最终也遗传给了 Spring Security。

Acegi Security 最终被并入 Spring Security 项目中，并于 2008 年 4 月发布了改名后的第一个版本 Spring Security 2.0.0，截止本书写作时， Spring Security 的最新版本已经到了 5.3.4。

和 Shiro 相比， Spring Security 重量级并且配置烦琐， 直至今天，依然有人以此为理由而拒绝了解 Spring Security。其实，自从 Spring Boot 推出后， 就彻底颠覆了传统了 JavaEE 开发，自动化配置让许多事情变得非常容易，包括 Spring Security 的配置。 在一个 Spring Boot 项目中，我们甚至只需要引入一个依赖，不需要任何额外配置，项目的所有接口就会被自动保护起来了。 在 Spring Cloud 中，很多涉及安全管理的问题，也是一个 Spring Security 依赖两行配置
就能搞定，在和 Spring 家族的产品一起使用时， Spring Security 的优势就非常明显了。

因此，在微服务时代，我们不需要纠结要不要学习 Spring Security，我们要考虑的是如何快速掌握 Spring Security，并且能够使用 Spring Security 实现我们微服务的安全管理。  

## 1.2 Spring Security 核心功能  

对于一个安全管理框架而言，无论是 Shiro 还是 Spring  Security，最核心的功能，无非就
是如下两方面：
●     认证
●     授权
通俗点说，认证就是身份验证（你是谁？），授权就是访问控制（你可以做什么？）。

### 1.2.1 认证  

