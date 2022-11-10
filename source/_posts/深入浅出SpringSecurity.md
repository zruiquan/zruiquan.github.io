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

Spring Security 支持多种不同的认证方式， 这些认证方式有的是 Spring Security 自己提供的认证功能， 有的是第三方标准组织制订的。 Spring Security 集成的主流认证机制主要有如下几种：

* 表单认证。
* OAutli2.0 认证。
* SAML2.0 认证。
* CAS 认证。
* RemembeiMe 自动认证。
* JAAS 认证。
* OpenID 去中心化认证。
* Pre-Authentication Scenarios 认证
* X509 认证。
* HTTP Basic 认证。
* HTTP Digest 认证。

作为一个开放的平台， Spring Security 提供的认证机制不仅仅包括上而这些， 我们还可以通过引入第三方依赖来支持更多的认证方式， 同时， 如果这些认证方式无法满足我们的需求，我们也可以自定义认证逻辑， 特别是当我们和一些“老破旧” 的系统进行集成时， 自定义认证逻辑就显得非常重要了。  

### 1.2.2 授权  

无论釆用了上而哪种认证方式，都不影响在 Spring Security 中使用授权功能。SpringSecurity 支持基于 URL 的请求授权、支持方法访问授权、支持 SpELt方问控制、 支持域对象安全（ACL) ，同时也支持动态权限配置、支持 RBAC 权限模型等，总之，我们常见的权限管理需求，Spring Security 基本上都是支持的。  

### 1.2.3 其他  

在认证和授权这两个核心功能之外， Spring Security 还提供了很多安全管理的”周边功能“，这也是一个非常重要的特色。

大部分 Java 工程师都不是专业的 Web 安全工程师， 自己开发的安全管理框架可能会存在大大小小的安全漏洞。 而Spring Security 的强大之处在于， 即使你不了解很多网络攻击， 只要使用了 Spring Security， 它会帮助我们自动防御很多网络攻击， 例如 CSRF 攻击、 会话固定攻击等， 同时 Spring Security 还提供了HTTP 防火墙来拦截大量的非法请求。 由此可见， 研究  Spring Security, 也是研究常见的网络攻击以及防御策略。

对于大部分的 Java 项目无论是从经济性还是安全性来考虑， 使用 Spring Security 无疑是最佳方案。  

## 1.3 Spring Security 整体架构  

在具体学习 Spring Security 各种用法之前， 我们先介绍一下 Spring Security 中常见的概念，以及认证、 授权思路， 方便读者从整体上把握 Spring Security 架构， 这里涉及的所有组件， 在后而的章节中还会做详细介绍。  

### 1.3.1 认证和授权  

#### 1.3.1.1 认证

在 Spring Security 的架构设计中， 认证（ Authenticatiou ) 和授权（Authorization) 是分开的， 在本书后而的章节中读者可以看到， 无论使用什么样的认证方式， 都不会影响授权， 这是两个独立的存在， 这种独立带来的好处之一， 就是 Spring Security 可以非常方便地整合一些外部的认证方案.  

在 Spring Security 中 ， 用 户 的 认 证 信 息 主 要 由 Authentication 的 实 现 类 来 保 存 ，Authentication 接口定义如下：  

```java
public interface Authentication extends Principal, Serializable {
    Collection<? extends GrantedAuthority> getAothorities();
    Object getCredentials();
    Object getDetails();
    Object getPrincipal();
    boolean isAuthenticated();
    void setAuthenticated(boolean isAuthenticated);
}
```

这里接n中定义的方法如下：

* getAutliorities 方法： 用来获取用户的权限。
* getCredentials 方法： 用来荻取用户凭证， 一般来说就是密码。
* getDetails 方法： 用来获取用户携带的详细信息， 可能是当前请求之类等。
* getPiincipal 方法： 用来获取当前用户， 例如是一个用户名或者一个用户对象
* isAuthenticated: 当前用户是否认证成功。

当用户使用用户名/密 码登录或使用 Remember-me 登录时 ， 都 会 对 应 一 个 不 同 的Authentication 实例。  

 {% pdf pdf/深入浅出Spring Security.pdf %}
