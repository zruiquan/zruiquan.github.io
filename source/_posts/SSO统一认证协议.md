---
title: SSO统一认证协议
date: 2022-02-16 14:21:18
tags:
- SpringSecurity
categories:
- SpringSecurity
---

# OAuth2.0

### **OAuth2.0介绍**

OAuth（Open Authorization）是一个关于授权（authorization）的开放网络标准，允许用户授权第三方 应用访问他们存储在另外的服务提供者上的信息，而不需要将用户名和密码提供给第三方移动应用或分享他 们数据的所有内容。OAuth在全世界得到广泛应用，目前的版本是2.0版。

OAuth协议：[https://tools.ietf.org/html/rfc6749](https://link.zhihu.com/?target=https%3A//tools.ietf.org/html/rfc6749)

协议特点：

- 简单：不管是OAuth服务提供者还是应用开发者，都很易于理解与使用；
- 安全：没有涉及到用户密钥等信息，更安全更灵活；
- 开放：任何服务提供商都可以实现OAuth，任何软件开发商都可以使用OAuth；

### **应用场景**

- 原生app授权：app登录请求后台接口，为了安全认证，所有请求都带token信息，如果登录验证、 请求后台数据。
- 前后端分离单页面应用：前后端分离框架，前端请求后台数据，需要进行oauth2安全认证
- 第三方应用授权登录，比如QQ，微博，微信的授权登录。



![img](SSO统一认证协议/v2-67306ed5439e40a969ac00acf90a4dc6_720w.webp)



### 基本概念

- **Third-party application**：第三方应用程序，又称"客户端"（client），即例子中的"豆瓣"。
- **HTTP service**：HTTP服务提供商，简称"服务提供商"，即例子中的qq。
- **Resource Owner**：资源所有者，又称"用户"（user）。
- **User Agent**：用户代理，比如浏览器。
- **Authorization server**：授权服务器，即服务提供商专门用来处理认证授权的服务器。
- **Resource server**：资源服务器，即服务提供商存放用户生成的资源的服务器。它与授权服务器，可以是同一台服务器，也可以是不同的服务器。

OAuth的作用就是让"客户端"安全可控地获取"用户"的授权，与"服务提供商"进行交互。

### **优缺点**

优点：

- 更安全，客户端不接触用户密码，服务器端更易集中保护
- 广泛传播并被持续采用
- 短寿命和封装的token
- 资源服务器和授权服务器解耦
- 集中式授权，简化客户端
- HTTP/JSON友好，易于请求和传递token
- 考虑多种客户端架构场景
- 客户可以具有不同的信任级别

缺点：

- 协议框架太宽泛，造成各种实现的兼容性和互操作性差
- 不是一个认证协议，本身并不能告诉你任何用户信息。

### Spring Security

Spring Security是一个能够为基于Spring的企业应用系统提供声明式的安全访问控制解决方案的**安全框架**。Spring Security 主要实现了**Authentication**（认证，解决who are you? ） 和 **Access Control**（访问控制，也就是what are you allowed to do？，也称为**Authorization**）。Spring Security在架构上将认证与授权分离，并提供了扩展点。

**认证（Authentication）** ：用户认证就是判断一个用户的身份是否合法的过程，用户去访问系统资源时系统要求验证用户的身份信息，身份合法方可继续访问，不合法则拒绝访问。常见的用户身份认证方式有：用户名密码登录，二维码登录，手机短信登录，指纹认证等方式。

**授权（Authorization）**： 授权是用户认证通过根据用户的权限来控制用户访问资源的过程，拥有资源的访问权限则正常访问，没有权限则拒绝访问。

将OAuth2和Spring Security集成，就可以得到一套完整的安全解决方案。我们可以通过Spring Security OAuth2构建一个授权服务器来验证用户身份以提供access_token，并使用这个access_token来从资源服务器请求数据。

### Spring Security + Oauth2.0**整体架构**



![img](SSO统一认证协议/v2-64f9510b46b6b3b9a0f41296db3f3f70_720w.webp)



- Authorize Endpoint ：授权端点，进行授权
- Token Endpoint ：令牌端点，经过授权拿到对应的Token
- Introspection Endpoint ：校验端点，校验Token的合法性
- Revocation Endpoint ：撤销端点，撤销授权



![img](SSO统一认证协议/v2-d0ea85d5a25c5bfb82ae0decdfd8a4a4_720w.webp)



1. 用户访问,此时没有Token。Oauth2RestTemplate会报错，这个报错信息会被Oauth2ClientContextFilter捕获并重定向到授权服务器。
2. 授权服务器通过Authorization Endpoint进行授权，并通过AuthorizationServerTokenServices生成授权码并返回给客户端。
3. 客户端拿到授权码去授权服务器通过Token Endpoint调用AuthorizationServerTokenServices生成Token并返回给客户端
4. 客户端拿到Token去资源服务器访问资源，一般会通过Oauth2AuthenticationManager调用ResourceServerTokenServices进行校验。校验通过可以获取资源。

### OAuth2.0模式

### 运行流程

OAuth 2.0的运行流程如下图，摘自RFC 6749。



![img](SSO统一认证协议/v2-fccac16d440911894f2ff356064563e9_720w.webp)



（A）用户打开客户端以后，客户端要求用户给予授权。

（B）用户同意给予客户端授权。

（C）客户端使用上一步获得的授权，向认证服务器申请令牌。

（D）认证服务器对客户端进行认证以后，确认无误，同意发放令牌。 

（E）客户端使用令牌，向资源服务器申请获取资源。

（F）资源服务器确认令牌无误，同意向客户端开放资源。

引入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-oauth2</artifactId>
</dependency>    
```

### 授权模式

客户端必须得到用户的授权（authorization grant），才能获得令牌（access token）。OAuth 2.0一共分成四种授权类型（authorization grant）

- 授权码模式（authorization code）
- 简化模式（implicit）
- 密码模式（resource owner password credentials）
- 客户端模式（client credentials）

授权码模式和密码模式比较常用。

第三方应用申请令牌之前，都必须先到系统备案，说明自己的身份，然后会拿到两个身份识别码：客户端 ID（client ID）和客户端密钥（client secret）。这是为了防止令牌被滥用，没有备案过的第三方应用，是不会拿到令牌的。

#### **授权码模式**

**授权码（authorization code）方式，指的是第三方应用先申请一个授权码，然后再用该码获取令牌。**

这种方式是最常用的流程，安全性也最高，它适用于那些有后端的 Web 应用。授权码通过前端传送，令牌则是储存在后端，而且所有与资源服务器的通信都在后端完成。这样的前后端分离，可以避免令牌泄漏。

适用场景：目前主流的第三方验证都是采用这种模式

![img](SSO统一认证协议/v2-2c1e8a6347274092c0c455a0d1130a5f_720w.webp)



主要流程：

（A）用户访问客户端，后者将前者导向授权服务器。

（B）用户选择是否给予客户端授权。

（C）用户给予授权，授权服务器将用户导向客户端事先指定的"重定向URI"（redirection URI），同时附上一个授权码。

（D）客户端收到授权码，附上早先的"重定向URI"，向授权服务器申请令牌。这一步是在客户端的后台的服务器上完成的，对用户不可见。

（E）授权服务器核对了授权码和重定向URI，确认无误后，向客户端发送访问令牌（access token）和更新令牌（refresh token）。



#### **配置授权服务器**

注意：实际项目中clinet_id 和client_secret 是配置在数据库中，省略spring security相关配置，可以参考

```java
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private TokenStore tokenStore;

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints
                .tokenStore(tokenStore)  //指定token存储到redis
                .allowedTokenEndpointRequestMethods(HttpMethod.GET,HttpMethod.POST); //支持GET,POST请求

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {

        // 基于内存模式，这里只是演示，实际生产中应该基于数据库
        clients.inMemory()
                //配置client_id
                .withClient("client")
                //配置client_secret
                .secret(passwordEncoder.encode("123123"))
                //配置访问token的有效期
                .accessTokenValiditySeconds(3600)
                //配置刷新token的有效期
                .refreshTokenValiditySeconds(864000)
                //配置redirect_uri，用于授权成功后跳转
                .redirectUris("http://www.baidu.com")
                //配置申请的权限范围
                .scopes("all")
                //配置grant_type，表示授权类型  authorization_code: 授权码
                .authorizedGrantTypes("authorization_code");


    }
}
```

客户端信息，和授权码都是存储在了内存中，一旦认证服务宕机，那客户端的认证信息也随之消失，而且客户端信息是在程序中写死的，维护起来及不方便，每次修改都需要重启服务，如果向上述信息都存于数据库中便可以解决上面的问题，其中数据我们可以自定义存到 noSql 或其他数据库中。



**数据库存储模式**

首先在数据库中新建存储客户端信息，及授权码的表：

```sql
#客户端信息
CREATE TABLE `oauth_client_details`  (
  `client_id` varchar(128) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  `resource_ids` varchar(256) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `client_secret` varchar(256) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `scope` varchar(256) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `authorized_grant_types` varchar(256) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `web_server_redirect_uri` varchar(256) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `authorities` varchar(256) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `access_token_validity` int(11) NULL DEFAULT NULL,
  `refresh_token_validity` int(11) NULL DEFAULT NULL,
  `additional_information` varchar(4096) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `autoapprove` varchar(256) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  PRIMARY KEY (`client_id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of oauth_client_details
-- ----------------------------
INSERT INTO `oauth_client_details` VALUES ('client', NULL, '$2a$10$CE1GKj9eBZsNNMCZV2hpo.QBOz93ojy9mTd9YQaOy8H4JAyYKVlm6', 'all', 'authorization_code,password,refresh_token,client_credentials,implicit', 'http://www.baidu.com', NULL, 3600, 864000, NULL, NULL);

#授权码
CREATE TABLE `oauth_code`  (
  `code` varchar(256) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,
  `authentication` blob NULL
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;
```

引入依赖

```java
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.3.2</version>
</dependency>

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>

<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.6</version>
</dependency>

/**
 * 授权服务器
 *
 * 授权码模式基于数据库
 */
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig1 extends AuthorizationServerConfigurerAdapter {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private TokenStore tokenStore;

    @Autowired
    private DataSource dataSource;

    @Autowired
    private AuthorizationCodeServices authorizationCodeServices;

    @Autowired
    @Qualifier("jdbcClientDetailsService")
    private ClientDetailsService clientDetailsService;

    @Bean("jdbcClientDetailsService")
    public ClientDetailsService clientDetailsService() {
        JdbcClientDetailsService clientDetailsService = new JdbcClientDetailsService(dataSource);
        clientDetailsService.setPasswordEncoder(passwordEncoder);
        return clientDetailsService;
    }

    //设置授权码模式的授权码如何存取
    @Bean
    public AuthorizationCodeServices authorizationCodeServices() {
        return new JdbcAuthorizationCodeServices(dataSource);
    }




    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints
                .tokenStore(tokenStore)  //指定token存储到redis
                .authorizationCodeServices(authorizationCodeServices)//授权码服务
                .allowedTokenEndpointRequestMethods(HttpMethod.GET,HttpMethod.POST); //支持GET,POST请求


    }
    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        //允许表单认证
        security
                .tokenKeyAccess("permitAll()")                    //oauth/token_key是公开
                .checkTokenAccess("permitAll()")                  //oauth/check_token公开
                .allowFormAuthenticationForClients();
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        //授权码模式
        //http://localhost:8080/oauth/authorize?response_type=code&client_id=client&redirect_uri=http://www.baidu.com&scope=all
        // 简化模式
//        http://localhost:8080/oauth/authorize?response_type=token&client_id=client&redirect_uri=http://www.baidu.com&scope=all

        clients.withClientDetails(clientDetailsService);


    }
}
```

#### **配置资源服务器**

```java
@Configuration
@EnableResourceServer
public class ResourceServiceConfig extends ResourceServerConfigurerAdapter {

    @Override
    public void configure(HttpSecurity http) throws Exception {

        http.authorizeRequests()
                .anyRequest().authenticated()
                .and().requestMatchers().antMatchers("/user/**");

    }

}
```

#### 配置 spring security

```java
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }


    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin().permitAll()
                .and().authorizeRequests()
                .antMatchers("/oauth/**").permitAll()
                .anyRequest().authenticated()
                .and().logout().permitAll()
                .and().csrf().disable();
    }
}

@Service
public class UserService implements UserDetailsService {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        String password = passwordEncoder.encode("123456");
        return new User("test",password, AuthorityUtils.commaSeparatedStringToAuthorityList("admin"));
    }
}
```

1、A网站（客户端）提供一个链接，用户点击后就会跳转到 B （授权服务器）网站，授权用户数据给 A 网站使用。

```properties
 127.0.0.1:9999/oauth/authorize?   
 response_type=code&            # 表示授权类型，必选项，此处的值固定为"code"
 client_id=CLIENT_ID&           # 表示客户端的ID，必选项
 redirect_uri=CALLBACK_URL&     # redirect_uri：表示重定向URI，可选项
 scope=read                     # 表示申请的权限范围，可选项 read只读  
```

http://localhost:8080/oauth/authorize?response_type=code&client_id=client&redirect_uri=http://www.baidu.com&scope=all

2、用户跳转后，B 网站会要求用户登录，然后询问是否同意给予 A 网站授权。用户表示同意，这时 B 网站就会跳回redirect_uri参数指定的网址。跳转时，会传回一个授权码

登录：



![img](SSO统一认证协议/v2-8d48abd15e424900ea6a6d780607cf63_720w.webp)



授权：



![img](SSO统一认证协议/v2-771b132d02c42ba3eccfad00fcdea176_720w.webp)



选择 authorize ，获取授权码，浏览器返回：[https://www.baidu.com/?code=PVpEEw](https://link.zhihu.com/?target=https%3A//www.baidu.com/%3Fcode%3DPVpEEw)

```properties
https://a.com/callback?code=AUTHORIZATION_CODE    #code参数就是授权码                   
```

如果使用数据库模式：



![img](SSO统一认证协议/v2-e465f211c92d93890334ed2ad1e1d337_720w.webp)



3、A 网站拿到授权码以后，就可以在后端，向 B 网站请求令牌。 用户不可见，服务端行为

```properties
127.0.0.1:8080/oauth/token? 
client_id=CLIENT_ID& 
client_secret=CLIENT_SECRET&     # client_id和client_secret用来让 B 确认 A 的身份,client_secret参数是保密的，因此只能在后端发请求 
grant_type=authorization_code&   # 采用的授权方式是授权码
code=AUTHORIZATION_CODE&         # 上一步拿到的授权码 
redirect_uri=CALLBACK_URL        # 令牌颁发后的回调网址         
```

4、B 网站收到请求以后，就会颁发令牌。具体做法是向redirect_uri指定的网址，返回数据。

```json
 {    
   "access_token":"ACCESS_TOKEN",     # 令牌
   "token_type":"bearer",
   "expires_in":2592000,
   "refresh_token":"REFRESH_TOKEN",
   "scope":"read",
   "uid":100101,
   "info":{...}
 }
```



![img](SSO统一认证协议/v2-8443ed6dfe341142d0104ebf9d0d4378_720w.webp)



此时redis中会存储token



![img](SSO统一认证协议/v2-295117bf22b768f44623556f66ddec7a_720w.webp)



### **密码模式**

如果你高度信任某个应用，RFC 6749 也允许用户把用户名和密码，直接告诉该应用。该应用就使用你的密码，申请令牌，这种方式称为"密码式"（password）。

在这种模式中，用户必须把自己的密码给客户端，但是客户端不得储存密码。这通常用在用户对客户端高度信任的情况下，比如客户端是操作系统的一部分，或者由一个著名公司出品。而授权服务器只有在其他授权模式无法执行的情况下，才能考虑使用这种模式。

适用场景：公司搭建的授权服务器



![img](SSO统一认证协议/v2-9f26138da24bba989525e73cb59589bb_720w.webp)



（A）用户向客户端提供用户名和密码。

（B）客户端将用户名和密码发给授权服务器，向后者请求令牌。

（C）授权服务器确认无误后，向客户端提供访问令牌。

#### 配置 spring security

增加AuthenticationManager

```java
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }

    @Autowired
    private UserService userService;

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        // 获取用户信息
        auth.userDetailsService(userService);
    }


    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.formLogin().permitAll()
                .and().authorizeRequests()
                .antMatchers("/oauth/**").permitAll()
                .anyRequest()
                .authenticated()
                .and().logout().permitAll()
                .and().csrf().disable();
    }


    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        // oauth2 密码模式需要拿到这个bean
        return super.authenticationManagerBean();
    }
}


@Service
public class UserService implements UserDetailsService {

    @Autowired
    @Lazy
    private PasswordEncoder passwordEncoder;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        String password = passwordEncoder.encode("123456");
        return new User("jack",password, AuthorityUtils.commaSeparatedStringToAuthorityList("admin"));
    }
}

@Configuration
public class RedisConfig {
    @Autowired
    private RedisConnectionFactory redisConnectionFactory;
    @Bean
    public TokenStore tokenStore(){
        return new RedisTokenStore(redisConnectionFactory);
    }
}
```

#### 配置授权服务器

```java
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig2 extends AuthorizationServerConfigurerAdapter {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private AuthenticationManager authenticationManagerBean;

    @Autowired
    private UserService userService;

    @Autowired
    private TokenStore tokenStore;


    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints.authenticationManager(authenticationManagerBean) //使用密码模式需要配置
                .tokenStore(tokenStore)  //指定token存储到redis
                .reuseRefreshTokens(false)  //refresh_token是否重复使用
                .userDetailsService(userService) //刷新令牌授权包含对用户信息的检查
                .allowedTokenEndpointRequestMethods(HttpMethod.GET,HttpMethod.POST); //支持GET,POST请求
    }

    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        //允许表单认证
        security.allowFormAuthenticationForClients();
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {

        clients.inMemory()
                //配置client_id
                .withClient("client")
                //配置client-secret
                .secret(passwordEncoder.encode("123123"))
                //配置访问token的有效期
                .accessTokenValiditySeconds(3600)
                //配置刷新token的有效期
                .refreshTokenValiditySeconds(864000)
                //配置redirect_uri，用于授权成功后跳转
                .redirectUris("http://www.baidu.com")
                //配置申请的权限范围
                .scopes("all")
                /**
                 * 配置grant_type，表示授权类型
                 * authorization_code: 授权码模式
                 * implicit: 简化模式
                 * password： 密码模式
                 * client_credentials: 客户端模式
                 * refresh_token: 更新令牌
                 */
                .authorizedGrantTypes("authorization_code","implicit","password","client_credentials","refresh_token");
    }


}
```

如需支持数据库模式，只需要把授权服务器在授权码模式的基础上增加AuthenticationManager，关键代码如下：

```java
//AuthorizationServerConfig1
@Autowired
private AuthenticationManager authenticationManagerBean;

@Override
public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
    endpoints.authenticationManager(authenticationManagerBean) //使用密码模式需要配置
    .tokenStore(tokenStore)  //指定token存储到redis
    .authorizationCodeServices(authorizationCodeServices)//授权码服务
    .allowedTokenEndpointRequestMethods(HttpMethod.GET,HttpMethod.POST); //支持GET,POST请求


}
```

获取token：

```properties
https://oauth.b.com/oauth/token?
  grant_type=password&       # 授权方式是"密码式"
  username=USERNAME&
  password=PASSWORD&
  client_id=CLIENT_ID&
  client_secret=123123&
  scope=all
```



![img](SSO统一认证协议/v2-7ddf3b5a046a6328ccd4f877b271d591_720w.webp)



### **客户端模式**

客户端模式（Client Credentials Grant）指客户端以自己的名义，而不是以用户的名义，向"服务提供商"进行 授权。

**适用于没有前端的命令行应用，即在命令行下请求令牌。**一般用来提供给我们完全信任的服务器端服务。



![img](SSO统一认证协议/v2-4e3a76b4df3cc5c699ee1e6c79fd7be9_720w.webp)



它的步骤如下：

（A）客户端向授权服务器进行身份认证，并要求一个访问令牌。

（B）授权服务器确认无误后，向客户端提供访问令牌。

A 应用在命令行向 B 发出请求。

```properties
https://oauth.b.com/token? 
grant_type=client_credentials& 
client_id=CLIENT_ID& 
client_secret=CLIENT_SECRET              
```

#### 配置授权服务器

在grant_type增加client_credentials来支持客户端模式。

```java
clients.inMemory()
        //配置client_id
        .withClient("client")
        //配置client-secret
        .secret(passwordEncoder.encode("123123"))
        //配置访问token的有效期
        .accessTokenValiditySeconds(3600)
        //配置刷新token的有效期
        .refreshTokenValiditySeconds(864000)
        //配置redirect_uri，用于授权成功后跳转
        .redirectUris("http://www.baidu.com")
        //配置申请的权限范围
        .scopes("all")
        /**
         * 配置grant_type，表示授权类型
         * authorization_code: 授权码模式
         * implicit: 简化模式
         * password： 密码模式
         * client_credentials: 客户端模式
         * refresh_token: 更新令牌
         */
        .authorizedGrantTypes("authorization_code","implicit","password","client_credentials","refresh_token");
```

获取令牌：



![img](SSO统一认证协议/v2-23f4d7129d2701d105fe82fdb1fcb4e5_720w.webp)



### **简化(隐式)模式**

有些 Web 应用是纯前端应用，没有后端。这时就不能用上面的方式了，必须将令牌储存在前端。**RFC 6749 就规定了第二种方式，允许直接向前端颁发令牌，这种方式没有授权码这个中间步骤，所以称为（授权码）"隐藏式"（implicit）**

简化模式不通过第三方应用程序的服务器，直接在浏览器中向授权服务器申请令牌，跳过了"授权码"这个步骤，所有步骤在浏览器中完成，令牌对访问者是可见的，且客户端不需要认证。

这种方式把令牌直接传给前端，是很不安全的。因此，只能用于一些安全要求不高的场景，并且令牌的有效期必须非常短，通常就是会话期间（session）有效，浏览器关掉，令牌就失效了。

![img](SSO统一认证协议/v2-e70efba7781be74052111a207a03f5e3_720w.webp)



它的步骤如下：

（A）客户端将用户导向授权服务器。

（B）用户决定是否给于客户端授权。

（C）假设用户给予授权，授权服务器将用户导向客户端指定的"重定向URI"，并在URI的Hash部分包含了访问令牌。

（D）浏览器向资源服务器发出请求，其中不包括上一步收到的Hash值。

（E）资源服务器返回一个网页，其中包含的代码可以获取Hash值中的令牌。

（F）浏览器执行上一步获得的脚本，提取出令牌。

（G）浏览器将令牌发给客户端。

A 网站提供一个链接，要求用户跳转到 B 网站，授权用户数据给 A 网站使用。

```properties
https://b.com/oauth/authorize?
>   response_type=token&          # response_type参数为token，表示要求直接返回令牌
>   client_id=CLIENT_ID&
>   redirect_uri=CALLBACK_URL&
>   scope=read
```

用户跳转到 B 网站，登录后同意给予 A 网站授权。这时，B 网站就会跳回redirect_uri参数指定的跳转网址，并且把令牌作为 URL 参数，传给 A 网站。

```properties
https://a.com/callback#token=ACCESS_TOKEN     #token参数就是令牌，A 网站直接在前端拿到令牌。 
```

#### 配置授权服务器

只需要在配置grant_type增加implicit

```java
clients.inMemory()
        //配置client_id
        .withClient("client")
        //配置client-secret
        .secret(passwordEncoder.encode("123123"))
        //配置访问token的有效期
        .accessTokenValiditySeconds(3600)
        //配置刷新token的有效期
        .refreshTokenValiditySeconds(864000)
        //配置redirect_uri，用于授权成功后跳转
        .redirectUris("http://www.baidu.com")
        //配置申请的权限范围
        .scopes("all")
        //配置grant_type，表示授权类型 授权码：authorization_code implicit: 简化模式
        .authorizedGrantTypes("authorization_code","implicit");
```

http://localhost:8080/oauth/authorize?client_id=client&response_type=token&scope=all&redirect_uri=http://www.baidu.com

登录之后进入授权页面，确定授权后浏览器会重定向到指定路径，并以Hash的形式存放在重定向uri的fargment中：



![img](SSO统一认证协议/v2-057af015eeab9ddf884e51fe929efbee_720w.webp)



如果想要支持数据库模式，配置同授权码模式一样，只需要在oauth_client_details表的authorized_grant_types配置上implicit即可。

```sql
INSERT INTO `oauth`.`oauth_client_details`(`client_id`, `resource_ids`, `client_secret`, `scope`, `authorized_grant_types`, `web_server_redirect_uri`, `authorities`, `access_token_validity`, `refresh_token_validity`, `additional_information`, `autoapprove`) VALUES ('gateway', NULL, '$2a$10$CE1GKj9eBZsNNMCZV2hpo.QBOz93ojy9mTd9YQaOy8H4JAyYKVlm6', 'all', 'authorization_code,password,refresh_token,implicit', 'http://www.baidu.com', NULL, 3600, 864000, NULL, NULL);
```

### **令牌的使用**

A 网站拿到令牌以后，就可以向 B 网站的 API 请求数据了。

此时，每个发到 API 的请求，都必须带有令牌。具体做法是在请求的头信息，加上一个Authorization字段，令牌就放在这个字段里面。



![img](SSO统一认证协议/v2-3b210a561d39ac8193b40d04179c6f37_720w.webp)



也可以添加请求参数access_token请求数据

```properties
localhost/user/getCurrentUser?access_token=xxxxxxxxxxxxxxxxxxxxxxxxxxx
```



![img](SSO统一认证协议/v2-4d6567cf4b86fd1236483e50062eae0a_720w.webp)



### **更新令牌**

令牌的有效期到了，如果让用户重新走一遍上面的流程，再申请一个新的令牌，很可能体验不好，而且也没有必要。OAuth 2.0 允许用户自动更新令牌。



![img](SSO统一认证协议/v2-221564197e4555ad4f753962c075aedb_720w.webp)



具体方法是，B 网站颁发令牌的时候，一次性颁发两个令牌，一个用于获取数据，另一个用于获取新的令牌（refresh token 字段）。令牌到期前，用户使用 refresh token 发一个请求，去更新令牌。

```properties
https://b.com/oauth/token?
>   grant_type=refresh_token&    # grant_type参数为refresh_token表示要求更新令牌
>   client_id=CLIENT_ID&
>   client_secret=CLIENT_SECRET&
>   refresh_token=REFRESH_TOKEN    # 用于更新令牌的令牌
```



![img](SSO统一认证协议/v2-c4e70729b0b1bf4baacf0ec13b6b4927_720w.webp)



### **基于redis存储Token**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

### Redis配置类

```java
@Configuration
public class RedisConfig {
    @Autowired
    private RedisConnectionFactory redisConnectionFactory;
    @Bean
    public TokenStore tokenStore(){
        return new RedisTokenStore(redisConnectionFactory);
    }
}
```

在授权服务器配置中指定令牌的存储策略为Redis

```java
@Autowired
private TokenStore tokenStore;

@Override
public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
    endpoints.authenticationManager(authenticationManagerBean) //使用密码模式需要配置
        .tokenStore(tokenStore)  //指定token存储到redis
        .reuseRefreshTokens(false)  //refresh_token是否重复使用
        .userDetailsService(userService) //刷新令牌授权包含对用户信息的检查
        .allowedTokenEndpointRequestMethods(HttpMethod.GET,HttpMethod.POST); //支持GET,POST请求
}
```

### 总结

OAuth2.0 的授权简单理解其实就是获取令牌（token）的过程，OAuth 协议定义了四种获得令牌的授权方式（authorization grant ）：授权码（authorization-code）、简单式（implicit）、密码式（password）、客户端凭证（client credentials），一般常用的是授权码和密码模式。

# OAuth2.0

## 一、OAuth2.0是什么？

在OAuth2.0中“O”是Open的简称，表示“开放”的意思。Auth表示“授权”的意思，所以连起来OAuth表示“开放授权”的意思，它是一个关于授权（authorization）的开放网络标准，在全世界得到广泛应用。用一句话总结来说，OAuth2.0是一种授权协议。
OAuth允许用户授权第三方应用访问他存储在另外服务商里的各种信息数据，而这种授权不需要提供用户名和密码提供给第三方应用。

> eg:万视达App通过微信三方登录方式登录，并且获取用户微信的昵称和头像等资料，这个过程万视达App平台并没有输入用户名和密码，而是跳转到微信输入微信账号、密码，待微信授权认证成功后，将用户在微信存储的信息提供给万视达，然后万视达App利用微信提供的信息注册登录万视达账号。

```
总的来说：OAuth2.0这种授权协议，就是保证第三方应用只有在获得授权之后，才可以进一步访问授权者的数据。
```

## 二、三方指的是哪三方？

第三方应用：要获取用户（资源拥有者）存储在服务提供商里的资源的实例，通常是客户端，这里我们的万视达App就是第三方应用；

服务提供者：存储用户（资源拥有者）的资源信息的地方，向第三方应用提供用户相关信息的服务方，例如三方登录时的微信、QQ;

用户/资源拥有者：拿三方登录来说，指的是在微信或QQ中注册的用户；
![在这里插入图片描述](SSO统一认证协议/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5a6J5qKT5pmo,size_19,color_FFFFFF,t_70,g_se,x_16.png)

## 三、OAuth2.0的作用

### 1、解决的问题

用户登录应用时传统的方式是用户直接进入客户端应用的登录页面输入账号和密码，但有的用户感觉注册后再登录比较繁琐麻烦，于是用户选择使用相关社交账号（微信、QQ、微博）登录，这种方式通过用户的账号和密码去相关社交账号里获取数据存在严重的问题缺陷。

1. 如果第三方应用获取微信中的用户信息，那么你就把你的微信的账号和密码给第三应用。稍微有些安全意识，都不会这样做，这样很不安全；因此使用OAuth2.0可以避免向第三方暴露账号密码；
2. 第三方应用拥有了用户微信的所有权限，用户没法限制第三方应用获得授权的范围和有效期；因此OAuth2.0可以限制授权第三方应用获取微信部分功能，比如只可以获取用户信息，但不可以获取好友列表，有需求时再申请授权访问好友列表的权限；
3. 用户只有修改密码，才能收回赋予第三方应用权限，但是这样做会使得其他所有获得用户授权的第三方应用程序全部失效。
4. 只要有一个第三方应用程序被破解，就会导致用户密码泄漏，以及所有使用微信登录的应用的数据泄漏。

如果某个企业拥有多个应用系统平台，那么每个系统都需要设置一个账号密码，这种操作对于用户来说时繁琐麻烦的，没登录一个系统平台都要输入相应的账号密码，那么可不可以做一个平台，使任意用户可以在这个平台上注册了一个帐号以后，随后这个帐号和密码自动登记到这个平台中作为公共帐号，使用这个账号可以访问其它的已经授权访问的系统。在这种背景下，OAuth2.0协议就诞生了。

> OAuth的作用就是让"第三方应用"安全可控地获取"用户"的授权，与"服务商提供商"进行交互。本质是使用token令牌代替用户名密码。

### 2、项目应用场景

现在各大开放平台，如微信开放平台、腾讯开放平台、百度开放平台等大部分的开放平台都是使用的OAuth 2.0协议作为支撑

- 客户端App使用三方登录；
- 微信小程序登录授权；
- 多个服务的统一登录认证中心、内部系统之间受保护资源请求

## 四、OAuth2.0名词定义

> Third-party application *：第三方应用客户端，本文中指的是万视达客户端 HTTP
> service:服务提供商，本文中指的是微信 Resource Owner：用户/资源拥有者，本文指的是在微信中注册的用户
> Authorization server:认证服务器，在资源拥有者授权后,向客户端授权(颁发 access token)的服务器
> Resource server:资源服务器，务提供商存放用户生成的资源的服务器。它与认证服务器，可以是同一台服务器，也可以是不同的服务器。

- 第三方应用的作用：

扮演获取资源服务器数据信息的加色

- 用户/资源所有者的作用：

扮演只需要允许或拒绝第三方应用获得授权的角色

- 授权认证服务器的作用：

负责向第三方应用提供授权许可凭证code、令牌token等

- 资源服务器的作用：

提供给第三方应用注册接口，需要提供给第三方应用app_id和app_secret，提供给第三方应用开放资源的接口

## 五、OAuth2.0授权流程

### 1、OAuth的思路

> OAuth在"第三方应用"与"服务提供商"之间，设置了一个授权层。“第三方应用"不能直接登录"服务提供商”，只能登录授权层，以此将用户与客户端区分开来。"第三方应用"登录授权层所用的令牌（token），与用户的密码不同。用户可以在登录的时候，指定授权层令牌的权限范围和有效期。"第三方应用"登录授权层以后，"服务提供商"根据令牌的权限范围和有效期，向"第三方应用"开放用户储存的资料。

当用户登录了第三方应用后，会跳转到服务提供商获取一次性用户授权凭据）,再跳转回来交给第三方应用，第三方应用的服务器会把授权凭据和服务提供商给它的身份凭据一起交给服务方，这样服务方既可以确定第三方应用得到了用户服务授权，又可以确定第三方应用的身份是可以信任的，最终第三方应用可以顺利的获取到服务商提供的web API的接口数据。

### 2、交互流程

![在这里插入图片描述](SSO统一认证协议/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5a6J5qKT5pmo,size_20,color_FFFFFF,t_70,g_se,x_16.png)

> 1. 用户打第三方开客户端后，第三方客户端要访问服务提供方，要求用户给予授权；
> 2. 用户同意给予第三方客户端访问服务提供方的授权，并返回一个授权凭证Code；
> 3. 第三方应用使用第2步获取的授权凭证Code和身份认证信息（appid、appsecret）,向授权认证服务器申请授权令牌（token）；
> 4. 授权认证服务器验证三方客户端的授权凭证Code码和身份通过后，确认无误，同意授权，并返回一个资源访问的令牌（Access Token）；
> 5. 第三方客户端使用第4步获取的访问令牌Access Token）向资源服务器请求相关资源；
> 6. 资源服务器验证访问令牌（Access Token）通过后，将第三方客户端请求的资源返回，同意向客户端开放资源；

下面是时序图：
![在这里插入图片描述](SSO统一认证协议/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5a6J5qKT5pmo,size_20,color_FFFFFF,t_70,g_se,x_16-1668221202725-36.png)

## 六、OAuth2.0的授权模式

> 授权码模式（authorization code）
> 简化模式（implicit）
> 密码模式（resource owner passwordcredentials）
> 客户端模式（client credentials）

### 1、授权码模式

第三方应用先申请获取一个授权码，然后再使用该授权码获取令牌，最后使用令牌获取资源；授权码模式是工能最完整、流程最严密的授权模式。它的特点是通过客户端的后台服务器，与服务提供商的认证服务器进行交互。
![在这里插入图片描述](SSO统一认证协议/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5a6J5qKT5pmo,size_20,color_FFFFFF,t_70,g_se,x_16-1668221202725-37.png)

流程步骤

**A [步骤1，2]**：用户访问客户端，需要使用服务提供商（微信）的数据，触发客户端相关事件后，客户端拉起或重定向到服务提供商的页面或APP。
**B [步骤3]**：用户选择是否给予第三方客户端授权访问服务提供商（微信）数据的权限；
**C [步骤4]**：用户同意授权，授权认证服务器将授权凭证code码返回给客户端，并且会拉起应用或重定（redirect_uri）向到第三方网站；
**D [步骤5，6]**:客户端收到授权码后，将授权码code和ClientId或重定向URI发送给自己的服务器，客户端服务器再想认证服务器请求访问令牌access_token;
**E [步骤7，8]**：认证服务器核对了授权码和ClientId或重定向URI，确认无误后，向客户端服务器发送访问令牌（access_token）和更新令牌（refresh_token），然后客户端服务器再发送给客户端；
**F [步骤9，10]**：客户端持有access_token和需要请求的参数向客户端服务器发起资源请求，然后客户端服务器再向服务提供商的资源服务器请求资源（web API）；
**G [步骤11，12，13]**：服务提供商的资源服务器返回数据给客户端服务器，然后再回传给客户端使用；

**A步骤中客户端申请认证的URI，包含以下参数：**
URL示例：

```csharp
https://server.example.com/oauth/auth?response_type=code&client_id=CLIENT_ID&redirect_uri=CALLBACK_URL&scope=read   
1
```

参数解释：

- response_type：必选项，表示授权类型，此处的值固定为"code"，表示使用授权码模式；
- client_id：必选项，表示客户端的ID，第三方应用注册的id，身份认证；
- redirect_uri：可选项，表示重定向URI，接受或拒绝请求后的跳转网址； scope：可选项，表示申请的权限范围；
- state：表示客户端的当前状态，可以指定任意值，认证服务器会原封不动地返回这个值；

**C步骤中，服务器回应客户端的URI，包含以下参数：**
URL示例：

```csharp
https:///server.example.com/callback?code=AUTHORIZATION_CODE&state=xyz
1
```

**参数解释：**

- code：表示授权码，该码的有效期应该很短，通常设为10分钟，客户端只能使用该码一次，否则会被授权服务器拒绝。该码与客户端ID和重定向URI，是一一对应关系。
- state：如果客户端的请求中包含这个参数，认证服务器的回应也必须一模一样包含这个参数。

D步骤中，客户端向认证服务器申请令牌的HTTP请求，包含以下参数：
URL示例：
https://server.example.com/oauth/token?client_id=CLIENT_ID&client_secret=CLIENT_SECRET&grant_type=authorization_code&code=AUTHORIZATION_CODE&redirect_uri=CALLBACK_URL
复制代码
参数解释：

- client_id：客户端id，第三方应用在服务提供者平台注册的；
- client_secret：授权服务器的秘钥，第三方应用在服务提供者平台注册的；
- grant_type：值是AUTHORIZATION_CODE，表示采用的授权方式是授权码；
- code:表示上一步获得的授权码;
- redirect_uri:redirect_uri参数是令牌颁发后的回调网址,且必须与A步骤中的该参数值保持一致;

> 注：client_id参数和client_secret参数用来让 B 确认 A
> 的身份（client_secret参数是保密的，因此只能在后端发请求）

**E步骤中，认证服务器发送的HTTP回复，包含以下参数：**
服务提供者的平台收到请求以后，就会颁发令牌。具体做法是向redirect_uri指定的网址，发送一段 JSON 数据。

```csharp
{    
  "access_token":"ACCESS_TOKEN",
  "token_type":"bearer",
  "expires_in":2592000,
  "refresh_token":"REFRESH_TOKEN",
  "scope":"read",
  "uid":100101,
  "info":{...}
}
123456789
```

上面 JSON 数据中，access_token字段就是访问令牌。

### 2、简化（隐藏）模式

不需要获取授权码，第三方应用授权后授权认证服务器直接发送临牌给第三方应用，适用于静态网页应用，返回的access_token可见，access_token容易泄露且不可刷新。

> 简化模式不通过第三方应用程序的服务器，直接在客户端中向认证服务器申请令牌，跳过了"授权码"这个步骤，因此得名。所有步骤在客户端中完成，令牌对访问者是可见的，且客户端不需要认证。

![在这里插入图片描述](SSO统一认证协议/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5a6J5qKT5pmo,size_20,color_FFFFFF,t_70,g_se,x_16-1668221202725-38.png)

流程步骤

- **A [步骤1，2]**：用户访问客户端，需要使用服务提供商（微信）的数据，触发客户端相关事件后，客户端拉起或重定向到服务提供商的页面或APP。
- **B [步骤3]**：用户选择是否给予第三方客户端授权访问服务提供商（微信）数据的权限；
- **C [步骤4]**：用户同意授权，授权认证服务器将访问令牌access_token返回给客户端，并且会拉起应用或重定（redirect_uri）向到第三方网站；
- **D[步骤5]**：第三方客户端向资源服务器发出请求资源的请求；
- **E[步骤6，7]**：服务提供商的资源服务器返回数据给客户端服务器，然后再回传给客户端使用；

**A步骤中，客户端发出的HTTP请求，包含以下参数：**
URL示例：

```csharp
https://server.example.com/oauth/authorize?response_type=token&client_id=CLIENT_ID&redirect_uri=CALLBACK_URL&scope=read
1
```

参数解释：

- response_type：表示授权类型，此处的值固定为"token"，表示要求直接返回令牌，必选项；
- client_id：表示客户端的ID，第三方应用注册的id，用于身份认证，必选项；
- redirect_uri：表示重定向URI，接受或拒绝请求后的跳转网址，可选项；
- scope：表示申请的权限范围，可选项；
- state：表示客户端的当前状态，可以指定任意值，认证服务器会原封不动地返回这个值，可选项；

**C步骤中，认证服务器回应客户端的URI，包含以下参数：**
URL示例：

```csharp
https://server.example.com/cb#access_token=ACCESS_TOKEN&state=xyz&token_type=example&expires_in=3600
或
https://a.com/callback#token=ACCESS_TOKEN
123
```

参数解释：

- access_token：表示访问令牌，必选项。
- token_type：表示令牌类型，该值大小写不敏感，必选项。
- expires_in：表示过期时间，单位为秒。如果省略该参数，必须其他方式设置过期时间。
- scope：表示权限范围，如果与客户端申请的范围一致，此项可省略。
- state：如果客户端的请求中包含这个参数，认证服务器的回应也必须一模一样包含这个参数。

```
注：注意，令牌的位置是 URL 锚点（fragment），而不是查询字符串（querystring），这是因为 OAuth 2.0 允许跳转网址是 HTTP 协议，因此存在"中间人攻击"的风险，而浏览器跳转时，锚点不会发到服务器，就减少了泄漏令牌的风险。
```

### 3、密码模式

如果你高度信任某个应用，允许用户把用户名和密码，直接告诉该应用。该应用就使用你的密码，申请令牌，这种方式称为"密码式"

> 使用用户名/密码作为授权方式从授权服务器上获取令牌，一般不支持刷新令牌。这种方式风险很大，用户向客户端提供自己的用户名和密码。客户端使用这些信息，向"服务商提供商"索要授权。在这种模式中，用户必须把自己的密码给客户端，但是客户端不得储存密码。这通常用在用户对客户端高度信任的情况下，而认证服务器只有在其他授权模式无法执行的情况下，才能考虑使用这种模式。

流程步骤

- **A[步骤1，2]**：用户向第三方客户端提供，其在服务提供商那里注册的账户名和密码；
- **B[步骤3]**：客户端将用户名和密码发给认证服务器，向后者请求令牌access_token；
- **C[步骤4]**：授权认证服务器确认身份无误后，向客户端提供访问令牌access_token；

**B步骤中，客户端发出的HTTP请求，包含以下参数：**
URL示例：

```csharp
https://oauth.example.com/token?grant_type=password&username=USERNAME&password=PASSWORD&client_id=CLIENT_ID
```

参数解释：

- grant_type： 标识授权方式，这里的password表示"密码式，必选项；
- username： 表示用户名，必选项；
- password： 表示用户的密码，必选项；
- scope： 表示权限范围，可选项；

**认证服务器向客户端发送访问令牌，是一段 JSON 数据 **

```csharp
   {
     "access_token":"2YotnFZFEjr1zCsicMWpAA",
     "token_type":"example",
     "expires_in":3600,
     "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
     "example_parameter":"example_value"
   }
```

### 4、客户端模式

指客户端以自己的名义，而不是以用户的名义，向"服务提供商"进行认证。严格地说，客户端模式并不属于OAuth框架所要解决的问题。在这种模式中，用户直接向客户端注册，客户端以自己的名义要求"服务提供商"提供服务，其实不存在授权问题。
![在这里插入图片描述](SSO统一认证协议/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5a6J5qKT5pmo,size_20,color_FFFFFF,t_70,g_se,x_16-1668221202725-39.png)

主要是第2步和第3步：

- 客户端向授权认证服务器进行身份认证，并申请访问临牌（token）;
- 授权认证服务器验证通过后，向客户端提供访问令牌。

**A步骤中，客户端发出的HTTP请求，包含以下参数**
URL示例：

```csharp
https://oauth.example.com/token?grant_type=client_credentials&client_id=CLIENT_ID&client_secret=CLIENT_SECRET
```

参数解释：

- grant_type：值为client_credentials表示采用凭证式（客户端模式）
- client_id：客户端id，第三方应用在服务提供者平台注册的，用于身份认证；
- client_secret：授权服务器的秘钥，第三方应用在服务提供者平台注册的，用于身份认证；

服务提供者验证通过以后，直接返回令牌。
这种方式给出的令牌，是针对第三方应用的，而不是针对用户的，即有可能多个用户共享同一个令牌。

```csharp
{
  "access_token":"2YotnFZFEjr1zCsicMWpAA",
  "token_type":"example",
  "expires_in":3600,
  "example_parameter":"example_value"
}
```

## 七、令牌的使用和更新

### 令牌的使用

客户端拿到访问令牌后就可以通过web API向服务提供商的资源服务器请求资源了。
此时，每个发到 API 的请求，都必须带有令牌。具体做法是在请求的头信息，加上一个Authorization字段，令牌就放在这个字段里面。

```bash
bash
curl -H "Authorization: Bearer ACCESS_TOKEN" \
"https://api.example.com"
```

### 令牌的更新

令牌的有效期到了，如果让用户重新走一遍上面的流程，再申请一个新的令牌，很可能体验不好，而且也没有必要。OAuth 2.0 允许用户自动更新令牌。
具体方法是，服务提供商平台颁发令牌的时候，一次性颁发两个令牌，一个用于获取数据的access_token，另一个用于获取新的令牌refresh_token。令牌到期前，用户使用 refresh_token 发一个请求，去更新令牌。
URL示例：

```csharp
https://server.example.com/oauth/token?grant_type=refresh_token&client_id=CLIENT_ID&client_secret=CLIENT_SECRET&refresh_token=REFRESH_TOKEN
```

参数解释：

- grant_type: 参数为refresh_token表示要求更新令牌;
- client_id：客户端id，第三方应用在服务提供者平台注册的，用于身份认证；
- client_secret：授权服务器的秘钥，第三方应用在服务提供者平台注册的，用于身份认证；
- refresh_token: 参数就是用于更新令牌的令牌

## 八、OAuth2.0和1.0的区别

> OAuth2.0的最大改变就是不需要临时token了，直接authorize生成授权code，用code就可以换取access
> token了，同时access
> token加入过期，刷新机制，为了安全，要求第三方的授权接口必须是https的。OAuth2.0不能向下兼容OAuth1.0版本，OAuth2.0使用Https协议，OAuth1.0使用Http协议；

- OAuth2.0去掉了签名，改用SSL确保安全性：OAuth1.0需要保证授权码和令牌token在传输的时候不被截取篡改，所以用了很多签名反复验证，但在OAuth2.0中是基于Https的。这里有个问题Https也有可能被中间人劫持；
- 令牌刷新：OAuth2.0的访问令牌是“短命的”，且有刷新令牌（OAuth1.0可以存储一年及以上）；
  -OAuth2.0所有的token不再有对应的secret存在，签名过程简洁，这也直接导致OAuth2.0不兼容老版本；
- 对于redirect_uri的校验，OAuth1.0中没有提到redirect_uri的校验，那么OAuth2.0中要求进行redirect_uri的校验；
- 授权流程区别：OAuth1.0授权流程太过单一，除了Web应用以外，对桌面、移动应用来说不够友好。而OAuth2.0提供了四中授权模式；

## 九、三方登录中OAuth2.0的使用

无论使用那种平台授权登录，都需要去相应的开放平台备案，申请appid、appsecret，来证明你的身份，第三方持凭此就有资格去请求该平台的接口。这是为了防止令牌被滥用，没有备案过的第三方应用，是不会拿到令牌的。
注：目前移动应用上微信登录只提供原生的登录方式，需要用户安装微信客户端才能配合使用,通过通用链接（Universal Links）实现第三方应用与微信，QQ之间的跳转。

### 1、三方登录如何做到的呢？

- 用户的授权信息在授权服认证务器中是有记录保存的，当用户第一次授权给第三方用用后，以后不需要再次授权；
- 授权令牌（token）是有实效性的， access_token 超时后，可以使用 refresh_token 进行刷新，refresh_token 拥有较长的有效期（30 天），当 refresh_token 失效的后，需要用户重新授权；
- 每个用户在资源服务器中都有一个唯一的ID，第三方应用可以将其存储起来并与本地用户系统一一对应起来；

### 2、微信三方登录OAuth2.0获取access_token流程

- 微信 OAuth2.0 授权登录让微信用户使用微信身份安全登录第三方应用或网站，在微信用户授权登录已接入微信 OAuth2.0 的第三方应用后，第三方可以获取到用户的接口调用凭证（access_token），通过 access_token 可以进行微信开放平台授权关系接口调用，从而可实现获取微信用户基本开放信息和帮助用户实现基础开放功能等。

微信 OAuth2.0 授权登录目前支持 authorization_code 模式，适用于拥有 server 端的应用授权。该模式整体流程为：

> 第三方发起微信授权登录请求，微信用户允许授权第三方应用后，微信会拉起应用或重定向到第三方网站，并且带上授权临时票据code参数；
> 通过code参数加上AppID和AppSecret等，通过API换取access_token；
> 通过access_token进行接口调用，获取用户基本数据资源或帮助用户实现基本操作。

- 获取 access_token 时序图：

![在这里插入图片描述](SSO统一认证协议/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5a6J5qKT5pmo,size_20,color_FFFFFF,t_70,g_se,x_16-1668221202725-40.png)

- access_token刷新机制

access_token 是调用授权关系接口的调用凭证，由于 access_token 有效期（目前为 2 个小时）较短，当 access_token 超时后，可以使用 refresh_token 进行刷新，access_token 刷新结果有两种：

> 若access_token已超时，那么进行refresh_token会获取一个新的access_token，新的超时时间
> 若access_token未超时，那么进行refresh_token不会改变access_token，但超时时间会刷新，相当于续期access_token

refresh_token 拥有较长的有效期（30 天），当 refresh_token 失效的后，需要用户重新授权。
具体流程看微信开放平台文档

## 总结

总的来说OAuth就是一种授权机制，数据的所有者告诉系统，统一授权第三方应用进入系统，获取部分数据。系统产生短期有实效和权限范围的令牌（token）给第三方应用，用来代替密码，供第三方使用。OAuth2.0授权的核心就是颁发访问令牌、使用访问令牌。也可以认为OAuth2.0是一个安全协议，按照OAuth2.0的规范来实施，就可以用来保护互联网中受保护资源。

# CAS

## CAS概念

CAS是Central Authentiction Service 的缩写，中央认证服务。CAS是Yale大学发起的开源项目，旨在为Web应用系统提供一种可靠的单点登录方法，CAS在2004年12月正式称为JA-SIG的一个项目。官网地址：[https://www.apereo.org/projects/cas](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.apereo.org%2Fprojects%2Fcas)。特点如下:

- 开源的企业级单点登录解决方案；
- 支持多种协议（CAS，SAML， WS-Federation，OAuth2，OpenID，OpenID Connect，REST）；
- CAS Client 多语言支持包括Java, .Net, PHP, Perl, Apache等；
- 支持三方授权登录（例如ADFS，Facebook，Twitter，SAML2 IdP等）的委派身份验证；
- 支持多因素身份验证（Duo Security，FIDO U2F，YubiKey，Google Authenticator，Microsoft Azure，Authy等）。

## CAS架构

![image-20221111171425736](SSO统一认证协议/image-20221111171425736.png)

如图所示，CAS分为：CAS Client和CAS Server。

- CAS Client：需要接入单点登录的应用系统；
- CAS Server：统一认证中心；

## CAS协议

CAS协议，是CAS项目默认的协议，是一种简单而强大的基于票证的协议。涉及到几个核心概念如下：

- TGT (Ticket Granting Ticket)：俗称大令牌，存储到浏览器Cookie(Name=TGC, Value=“TGT的值”)上，可以通过TGT签发ST，具备时效性，默认2小时；
- ST (Service Ticket)：俗称小令牌，CAS Client通过ST校验是否登录成功，并获取用户信息。ST只有一次有效，有效期默认10秒。

## CAS协议认证流程图

![正常单点登录流程](SSO统一认证协议/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2lzeW91bmdib3k=,size_16,color_FFFFFF,t_70.png)

上图分为三种不同场景的访问方式：

- 第一次访问某个接入CAS认证中心的应用系统（app1）：

  1. 用户请求应用系统app1，app1判断用户未登录，跳转到CAS认证中心登录页面；

  2. 用户输入账号密码提交到CAS认证中心，如果认证通过则走第3步；

  3. CAS认证中心在它的域名下面的Cookie设置TGC，将签发的ST作为请求参数重定向到app1；

  4. app1通过ST校验登录是否合法，并获取用户信息，存储到本地session；

  5. app1跳转到开始访问的页面。

     ![image-20221112113916990](SSO统一认证协议/image-20221112113916990.png)

- 第二次访问某个接入CAS认证中心的应用系统（app1）：由于app1已经登录过了，之后访问app1都不需要再跟CAS认证中心打交道了。

  ![image-20221112113953499](SSO统一认证协议/image-20221112113953499.png)

- 第一次访问其他接入CAS认证中心的应用系统（app2）：

  1. 用户请求应用系统app2，app2判断用户未登录，跳转到CAS认证中心登录页面；

  2. CAS认证中心验证之前生成的TGT(从Cookie的TGC获取)，如果该TGT有效，则签发ST重定向到app2；

  3. app2通过ST校验登录是否合法，并获取用户信息，存储到本地session；

  4. app2跳转到开始访问的页面。

     ![image-20221112114010668](SSO统一认证协议/image-20221112114010668.png)

## CAS Server认证逻辑

![img](SSO统一认证协议/webp-1668158117361-9.webp)

上图是作者从源码梳理出来的逻辑，如有错误之处，欢迎指正~

## 实战手写CAS Client

根据前面学习到的CAS登录原理，我们来实战手写CAS Client端，巩固下刚刚学到的知识，详情见代码：

1. 新建一个Spring Boot项目（自行实现）；
2. 引入包：

```xml
       <!--需要应用到该包XML工具类-->
        <dependency>
            <groupId>org.jasig.cas.client</groupId>
            <artifactId>cas-client-core</artifactId>
            <version>3.6.1</version>
        </dependency>
       <!--发起http请求-->
        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpclient</artifactId>
            <version>4.3.3</version>
        </dependency>
```

1. 配置文件

```undefined
server.servlet.context-path=/client
server.port=8080
```

1. 代码实现
    定义一个过滤器，用于登录、登出逻辑认证

```java
@Configuration
public class AppConfig {
    @Bean
    public FilterRegistrationBean registerAuthFilter() {
        FilterRegistrationBean registration = new FilterRegistrationBean();
        registration.setFilter(new AccessFilter());
        registration.addUrlPatterns("/*");
        registration.setName("accessFilter");
        registration.setOrder(Integer.MIN_VALUE);
        return registration;
    }
}

public class AccessFilter implements Filter {
    private static final Logger LOGGER = LoggerFactory.getLogger(AccessFilter.class);
    public static final String USER_INFO = "USER_INFO";
    public static final String CAS_URL = "https://127.0.0.1:8443/cas";
    public static final String SERVICE_URL = "http://127.0.0.1:8080/client/";
    /** 记录登录的session，退出时，可以获取session并销毁 key-ST, value-session */
    private static final Map<String, HttpSession> ST_SESSION_MAP = new ConcurrentHashMap<>();

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse,
                         FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest) servletRequest;
        HttpServletResponse response = (HttpServletResponse) servletResponse;

        String ticket = request.getParameter("ticket");
        // 校验ST
        if (!StringUtils.isEmpty(ticket) && "GET".equals(request.getMethod())) {
            this.validateTicket(ticket, request.getSession());
        }

        // 监听登出
        String logoutRequest = request.getParameter("logoutRequest");
        if (!StringUtils.isEmpty(logoutRequest) && "POST".equals(request.getMethod())) {
            LOGGER.info(logoutRequest);
            this.destroySession(logoutRequest);
            return;
        }

        // 未登录？则跳转CAS登录页面
        if (request.getSession().getAttribute(USER_INFO) == null) {
            response.sendRedirect(CAS_URL + "/login?service=" + SERVICE_URL);
            return;
        }
        filterChain.doFilter(request, response);
    }

    /**
     * 校验ST
     * @param ticket
     * @param session
     */
    private void validateTicket(String ticket, HttpSession session) {
        String result = null;
        try {
            // 向CAS发起ST校验，并获取用户信息
            result = HttpClientUtils.doGet(CAS_URL + "/serviceValidate?service=" + SERVICE_URL
                    + "&ticket=" + ticket);
            LOGGER.info(result);
        } catch (Exception e) {
            throw new RuntimeException("serviceValidate请求失败:" + e.getMessage(), e);
        }
        // 校验成功可以解析到user信息（XML格式）
        String user = XmlUtils.getTextForElement(result, "user");
        // 记录登录信息
        if (!StringUtils.isEmpty(user)) {
            session.setAttribute(USER_INFO, user);
            ST_SESSION_MAP.put(ticket, session);
        } else {
            throw new RuntimeException("校验ST失败");
        }
    }

    /**
     * 销毁session
     * @param logoutRequest
     */
    private void destroySession(String logoutRequest) {
        final String ticket = XmlUtils.getTextForElement(logoutRequest, "SessionIndex");
        if (CommonUtils.isBlank(ticket)) {
            return;
        }
        final HttpSession session = ST_SESSION_MAP.get(ticket);
        if (session != null) {
            session.invalidate();
        }
    }

}
```

相关工具类

```java
public abstract class HttpClientUtils {
    private static final Logger LOGGER = LoggerFactory.getLogger(HttpClientUtils.class);
    public static final String ENCODING = "UTF-8";
    private static final int CONNECT_TIMEOUT = 3000;
    private static final int SOCKET_TIMEOUT = 10000;
    private static final int CONNECTION_REQUEST_TIMEOUT = 3000;



    private static CloseableHttpClient httpClient;

    static {
        httpClient = createHttpClient();
    }

    private static CloseableHttpClient createHttpClient() {
        if (httpClient != null) {
            return httpClient;
        }
        try {
            SSLContext sslContext = new SSLContextBuilder().loadTrustMaterial(null, new TrustStrategy() {
                public boolean isTrusted(X509Certificate[] chain,
                                         String authType) throws CertificateException {
                    return true;
                }
            }).build();
            SSLConnectionSocketFactory sslsf = new SSLConnectionSocketFactory(sslContext,SSLConnectionSocketFactory.ALLOW_ALL_HOSTNAME_VERIFIER);
            HttpClientBuilder httpClientBuilder = HttpClients.custom();
            httpClientBuilder.setSSLSocketFactory(sslsf);
            httpClientBuilder.setMaxConnPerRoute(50);
            httpClientBuilder.setMaxConnTotal(150);
            RequestConfig.Builder configBuilder = RequestConfig.custom();
            configBuilder.setConnectTimeout(CONNECT_TIMEOUT);
            configBuilder.setSocketTimeout(SOCKET_TIMEOUT);
            configBuilder.setConnectionRequestTimeout(CONNECTION_REQUEST_TIMEOUT);
            configBuilder.setStaleConnectionCheckEnabled(true);
            httpClientBuilder.setDefaultRequestConfig(configBuilder.build());
            httpClient = httpClientBuilder.build();
        } catch (Exception ex) {
            LOGGER.error("create https client support fail:"+ ex.getMessage(), ex);
        }
        return httpClient;
    }

    public static String doGet(String url) throws Exception {
        URIBuilder uriBuilder = new URIBuilder(url);
        HttpGet httpGet = new HttpGet(uriBuilder.build());
        return getHttpClientResult(httpGet);
    }

    public static String getHttpClientResult(HttpRequestBase httpMethod) throws IOException{
        CloseableHttpResponse httpResponse = null;
        try {
            httpResponse = httpClient.execute(httpMethod);
            String content = "";
            if (httpResponse != null && httpResponse.getStatusLine() != null && httpResponse.getEntity() != null) {
                content = EntityUtils.toString(httpResponse.getEntity(), ENCODING);
            }
            return content;
        } finally {
            if (httpResponse != null) {
                httpResponse.close();
            }
        }
    }
}
```

控制层代码

```java
@Controller
public class BaseController {

    @RequestMapping("/")
    public String home() {
        return "redirect:/index";
    }

    @RequestMapping("/index")
    @ResponseBody
    public String index(HttpServletRequest request) {
        return "登录用户：" + request.getSession().getAttribute(AccessFilter.USER_INFO);
    }

    @RequestMapping("/logout")
    public void logout(HttpServletResponse response) throws IOException {
        response.sendRedirect(AccessFilter.CAS_URL + "/logout?service=" + AccessFilter.SERVICE_URL);
    }
}
```

# OpenID Connect

## OpenID Connect 协议入门指南

如果要谈单点登录和身份认证，就不得不谈OpenID Connect (OIDC)。最典型的使用实例就是使用Google账户登录其他应用，这一经典的协议模式，为其他厂商的第三方登录起到了标杆的作用，被广泛参考和使用。

## OpenID Connect简介

OpenID Connect是基于OAuth 2.0规范族的可互操作的身份验证协议。它使用简单的REST / JSON消息流来实现，和之前任何一种身份认证协议相比，开发者可以轻松集成。

OpenID Connect允许开发者验证跨网站和应用的用户，而无需拥有和管理密码文件。OpenID Connect允许所有类型的客户,包括基于浏览器的[JavaScript](https://link.jianshu.com/?t=http://lib.csdn.net/base/javascript)和本机移动应用程序,启动登录流动和接收可验证断言对登录用户的身份。

## OpenID的历史是什么？

OpenID Connect是OpenID的第三代技术。首先是原始的OpenID，它不是商业应用，但让行业领导者思考什么是可能的。OpenID 2.0设计更为完善，提供良好的安全性保证。然而，其自身存在一些设计上的局限性，最致命的是其中依赖方必须是网页，但不能是本机应用程序；此外它还要依赖XML，这些都会导致一些应用问题。

OpenID Connect的目标是让更多的开发者使用，并扩大其使用范围。幸运的是，这个目标并不遥远，现在有很好的商业和开源库来帮助实现身份验证机制。

## OIDC基础

简要而言，*OIDC*是一种安全机制，用于应用连接到身份认证服务器（Identity Service）获取用户信息，并将这些信息以安全可靠的方法返回给应用。

在最初，因为OpenID1/2经常和[OAuth协议](https://www.jianshu.com/p/6392420faf99)（一种授权协议）一起提及，所以二者经常被搞混。

* **OpenID **是*Authentication*，即认证，对用户的身份进行认证，判断其身份是否有效，也就是让网站知道“你是你所声称的那个用户”；

* **OAuth **是*Authorization*，即授权，在已知用户身份合法的情况下，经用户授权来允许某些操作，也就是让网站知道“你能被允许做那些事情”。
  由此可知，授权要在认证之后进行，只有确定用户身份只有才能授权。

> (身份验证)+ OAuth 2.0 = OpenID Connect

*OpenID Connect*是“认证”和“授权”的结合，因为其基于*OAuth*协议，所以*OpenID-Connect*协议中也包含了**client_id**、**client_secret**还有**redirect_uri**等字段标识。这些信息被保存在“身份认证服务器”，以确保特定的客户端收到的信息只来自于合法的应用平台。这样做是目的是为了防止*client_id*泄露而造成的恶意网站发起的*OIDC*流程。

举个例子。某个用户使用*Facebook*应用*“What online quiz best describes you?”* ，该应用可以通过*Facebook*账号登录，则你可以在应用中发起请求到“身份认证服务器”（也就是Facebook的服务器）请求登录。这时你会看到如下界面，询问是否授权。

![img](SSO统一认证协议/872.png)

在*OAuth*中，这些授权被称为**scope**。*OpenID-Connect*也有自己特殊的*scope*--**openid** ,它必须在第一次请求“身份鉴别服务器”（Identity Provider,简称IDP）时发送过去。

## OIDC流程

*OAuth2*提供了*Access Token*来解决授权第三方客户端访问受保护资源的问题；相似的，*OIDC*在这个基础上提供了*ID Token*来解决第三方客户端标识用户身份认证的问题。*OIDC*的核心在于在*OAuth2*的授权流程中，一并提供用户的身份认证信息（*ID-Token*）给到第三方客户端，*ID-Token*使用**JWT**格式来包装，得益于**JWT**（[JSON Web Token](https://link.jianshu.com/?t=http://www.cnblogs.com/linianhui/p/oauth2-extensions-protocol-and-json-web-token.html#auto_id_6)）的自包含性，紧凑性以及防篡改机制，使得*ID-Token*可以安全的传递给第三方客户端程序并且容易被验证。应有服务器，在验证*ID-Token*正确只有，使用*Access-Token*向*UserInfo*的接口换取用户的更多的信息。

由上述可知，*OIDC*是遵循*OAuth*协议流程，在申请*Access-Token*的同时，也返回了*ID-Token*来验证用户身份。

### 相关定义

- **EU**：End User，用户。

- **RP**：Relying Party ，用来代指*OAuth2*中的受信任的客户端，身份认证和授权信息的消费方；

- **OP**：OpenID Provider，有能力提供EU身份认证的服务方（比如*OAuth2*中的授权服务），用来为RP提供EU的身份认证信息；

- **ID-Token**：JWT格式的数据，包含EU身份认证的信息。

  ![image-20221111200104621](SSO统一认证协议/image-20221111200104621.png)

- **UserInfo Endpoint**：用户信息接口（受*OAuth2*保护），当RP使用*ID-Token*访问时，返回授权用户的信息，此接口必须使用*HTTPS*。

下面我们来看看*OIDC*的具体协议流程。
根据应用客户端的不同，*OIDC*的工作模式也应该是不同的。和*OAuth*类似，主要看是否客户端能保证*client_secret*的安全性。

如果是JS应用，其所有的代码都会被加载到浏览器而暴露出来，没有后端可以保证*client_secret*的安全性，则需要是使用**默认模式流程**(Implicit Flow)。

如果是传统的客户端应用，后端代码和用户是隔离的，能保证*client_secret*的不被泄露，就可以使用**授权码模式流程**（Authentication Flow）。

此外还有**混合模式流程**(Hybrid Flow)，简而言之就是以上二者的融合。

> OAuth2中还有*口令模式*和“应有访问模式”的方式来获取Access Token（关于*OAuth2*的内容，可以参见[OAuth2.0 协议入门指南](https://www.jianshu.com/p/6392420faf99)），为什么OIDC没有扩展这些方式呢?
> "口令模式"是需要用户提供账号和口令给RP的，既然都已经有用户名和口令了，就不需要在获取什么用户身份了。至于“应有访问模式”，这种方式不需要用户参与，也就无需要认证和获取用户身份了。这也能反映**授权和认证的差异**，以及只使用*OAuth2*来做身份认证的事情是远远不够的，也是不合适的。

### 总体流程

类似OAuth2.0，有一个总体流程和若干细分模式的流程，OpenID Connect协议总体流程为：

1. RP发起请求 
2. OP对用户鉴权并获取授权 
3. OP响应RP并带上ID Token和Access Token 
4. RP通过访问凭证向OP请求用户信息 
5. OP返回用户信息给RP

```
+--------+                                   +--------+
|        |                                   |        |
|        |---------(1) AuthN Request-------->|        |
|        |                                   |        |
|        |  +--------+                       |        |
|        |  |        |                       |        |
|        |  |  End-  |<--(2) AuthN & AuthZ-->|        |
|        |  |  User  |                       |        |
|   RP   |  |        |                       |   OP   |
|        |  +--------+                       |        |
|        |                                   |        |
|        |<--------(3) AuthN Response--------|        |
|        |                                   |        |
|        |---------(4) UserInfo Request----->|        |
|        |                                   |        |
|        |<--------(5) UserInfo Response-----|        |
|        |                                   |        |
+--------+                                   +--------+
```

### 客户鉴权

客户鉴权即OP对客户端进行鉴权，然后将鉴权结果返回给RP。鉴权流程有三种方式：授权码模式、隐藏模式、混合模式。如果了解OAuth2.0，对前两个模式一定不会模式，OpenID流程类似，而混合模式则是前两种模式的结合。具体OP采用什么模式，取决于RP请求时response_type给的值。

| response_type       | 采用的模式 |
| ------------------- | ---------- |
| code                | 授权码模式 |
| id_token            | 隐藏模式   |
| Id_token token      | 隐藏模式   |
| code id_token       | 混合模式   |
| code token          | 混合模式   |
| code id_token token | 混合模式   |

> 规律：id_token和token等同：只有id_token或/和token的使用隐藏模式，只有code的使用授权码模式，同时存在他们的则使用混合模式

### 授权码模式流程

![img](SSO统一认证协议/800.png)

![img](SSO统一认证协议/bb.png)

授权码模式流程和*OAuth*认证流程类似

总计八个步骤

1. RP准备用于鉴权请求的参数，其中附带*client_id*；
2. RP发送请求，给OP
3. OP对用户鉴权
4. OP收集用户的鉴权信息和授权信息
5. OP发送授权码给RP
6. RP使用授权码向一个端点换取访问凭证。协议称之为Token端点，但没说这个端点是不是由OP提供的。
7. RP收到访问凭证，包含ID Token、Access Token
8. 客户端验证ID Token，并从中提取用户的唯一标识。前面说过这是一个JWT，唯一标识就是subject identifier，RP使用Access-Token发送一个请求到*UserInfo EndPoint*； UserInfo EndPoint返回EU的Claims。

#### **授权码的请求与响应**

其中RP准备的请求参数包含OAuth2.0规定的所有字段，也包含一些额外的字段

- scope：必须。OIDC的请求必须包含值为“openid”的scope的参数。
- response_type：必选。同OAuth2。
- client_id：必选。同OAuth2。
- redirect_uri：必选。同OAuth2。
- state：推荐。同OAuth2。防止CSRF, XSRF

![image-20221111195643035](SSO统一认证协议/image-20221111195643035.png)

示例如下：

```properties
GET /authorize?
    response_type=code
    &scope=openid%20profile%20email
    &client_id=s6BhdRkqt3
    &state=af0ifjsldkj
    &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb HTTP/1.1
  Host: server.example.com
```

如果授权成功，得到的返回会包含code、state等参数，举个例子

```properties
  HTTP/1.1 302 Found
  Location: https://client.example.org/cb?
  	code=SplxlOBeZQQYbYS6WxSbIA
  	&state=af0ifjsldkj
```

如果授权失败，也会有相应的错误码，具体[参考手册](https://openid.net/specs/openid-connect-core-1_0.html#toc)的3.1.2.6

#### **凭证的请求与响应**

这个就完全和OAuth2.0一样了。

请求上主要包含：`grant_type`写死`authorization_code`、`code`填上一步获取的授权码、`redirect_uri`填上一步的重定向地址，举个例子

```properties
POST /token HTTP/1.1
Host: server.example.com
Content-Type: application/x-www-form-urlencoded
# 你可能注意到这里有个Basic鉴权，这是因为Client也是需要被验证的
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW

grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA&redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
```

响应上多出一个`id_token`字段，举例如下

```properties
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache

{
  "access_token": "SlAV32hkKG",
  "token_type": "Bearer",
  "refresh_token": "8xLOxBtZp8",
  "expires_in": 3600,
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjFlOWdkazcifQ.ewogImlzc
    yI6ICJodHRwOi8vc2VydmVyLmV4YW1wbGUuY29tIiwKICJzdWIiOiAiMjQ4Mjg5
    NzYxMDAxIiwKICJhdWQiOiAiczZCaGRSa3F0MyIsCiAibm9uY2UiOiAibi0wUzZ
    fV3pBMk1qIiwKICJleHAiOiAxMzExMjgxOTcwLAogImlhdCI6IDEzMTEyODA5Nz
    AKfQ.ggW8hZ1EuVLuxNuuIJKX_V8a_OMXzR0EHR9R6jgdqrOOF4daGU96Sr_P6q
    Jp6IcmD3HP99Obi1PRs-cwh3LO-p146waJ8IhehcwL7F09JdijmBqkvPeB2T9CJ
    NqeGpe-gccMg4vfKjkM8FcGvnzZUN4_KSP0aAp1tOJ1zZwgjxqGByKHiOtX7Tpd
    QyHE5lcMiKPXfEIQILVq0pc_E2DzL7emopWoaoZTF_m0_N0YzFC6g6EJbOEoRoS
    K5hoDalrcvRYLSrQAZZKflyuVCyixEoV9GfNQC3_osjzw2PAithfubEEBLuVVk4
    XUVrWOLrLl0nx7RkKU8NXNHq-rvKMzqg"
}
```

> 其中看起来一堆乱码的部分就是JWT格式的*ID-Token*。在RP拿到这些信息之后，需要对*id_token*以及*access_token*进行验证（具体的规则参见[http://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation](https://link.jianshu.com/?t=http://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation)和[http://openid.net/specs/openid-connect-core-1_0.html#ImplicitTokenValidation](https://link.jianshu.com/?t=http://openid.net/specs/openid-connect-core-1_0.html#ImplicitTokenValidation)）。至此，可以说用户身份认证就可以完成了，后续可以根据UserInfo EndPoint获取更完整的信息。
>
> 在上面的每一方，比如OP收到请求、RP收到响应，都会对得到的数据进行验证，我们都忽略了，不过这里重点将RP收到ID Token后的验证逻辑列出来
>
> 1. 依据JWT协议解密该Token 
> 2. iss字段必须匹配RP提前获取到的OP的issuer值
> 3. aud字段必须是RP在OP处注册时填写的client_id
> 4. 如果包含多个aud字段，则还要验证azp字段
> 5. 如果azp字段存在，则它的值必须是RP在OP处注册时填写的client_id
> 6. 如果ID Token是RP直接从OP处获取，没有走授权码这一步（比如走隐藏模式的流程），必须走JWS流程验证该JWT的签名
> 7. alg的值要么为默认的RS256，要么是RP在OP处注册时通过id_token_signed_response_alg字段指定的算法
> 8. exp所展示的时间必须比当前时间晚
> 9. iat可以用来识别签发时间过久的JWT，太久的可以拒掉，多久算久，这个取决于RP自己
> 10. nonce必须和请求授权码时对应上
> 11. acr必须和请求授权码时对应上
> 12. 接和auth_time、max_age，可以判断举例上一次时间是不是太久，从而决定是否需要重新发起鉴权请求

#### 安全令牌 ID-Token

上面提到过*OIDC*对*OAuth2*最主要的扩展就是提供了*ID-Token*。下面我们就来看看*ID-Token*的主要构成：

- **iss = Issuer Identifier**：必须。提供认证信息者的唯一标识。一般是Url的host+path部分；
- **sub = Subject Identifier**：必须。iss提供的EU的唯一标识；最长为255个ASCII个字符；
- **aud = Audience(s)**：必须。标识*ID-Token*的受众。必须包含*OAuth2*的client_id；
- **exp = Expiration time**：必须。*ID-Token*的过期时间；
- **iat = Issued At Time**：必须。JWT的构建的时间。
- **auth_time = AuthenticationTime**：EU完成认证的时间。如果RP发送认证请求的时候携带*max_age*的参数，则此Claim是必须的。
- **nonce**：RP发送请求的时候提供的随机字符串，用来减缓重放攻击，也可以来关联*ID-Token*和RP本身的Session信息。
- **acr = Authentication Context Class Reference**：可选。表示一个认证上下文引用值，可以用来标识认证上下文类。
- **amr = Authentication Methods References**：可选。表示一组认证方法。
- **azp = Authorized party**：可选。结合aud使用。只有在被认证的一方和受众（aud）不一致时才使用此值，一般情况下很少使用。

```
{
   "iss": "https://server.example.com",
   "sub": "24400320",
   "aud": "s6BhdRkqt3",
   "nonce": "n-0S6_WzA2Mj",
   "exp": 1311281970,
   "iat": 1311280970,
   "auth_time": 1311280969,
   "acr": "urn:mace:incommon:iap:silver"
  }
```

另外ID Token必须使用[JWT(JSON Web Token)](https://link.jianshu.com/?t=https://tools.ietf.org/html/rfc7519)进行签名和[JWE(JSON Web Encryption)](https://link.jianshu.com/?t=https://tools.ietf.org/html/rfc7516)加密，从而提供认证的完整性、不可否认性以及可选的保密性。关于**JWT**的更多内容，请参看[JSON Web Token - 在Web应用间安全地传递信息](https://link.jianshu.com/?t=http://blog.leapoahead.com/2015/09/06/understanding-jwt/)

### 隐藏模式流程

![img](SSO统一认证协议/900.png)

![img](SSO统一认证协议/bb-1668161051599-33.png)



隐藏流程和*OAuth*中的类似，只不过也是添加了*ID-Token*的相关内容。

这个模式就比较简单了

1. RP准备请求参数
2. RP发送请求
3. OP认证用户
4. OP获取用户的认证和授权信息
5. OP发送ID Token，可能还有Access Token给RP
6. RP验证ID Token，提取用户的标识

**请求**

请求参数和授权码模式基本一样，这里列出差别

- response_type：id_token，或者`id_token token`，差别是，如果加上token，步骤5会返回Access Token
- redirect_uri：处于安全考虑，这个地方必须使用HTTPS传输
- nonce：这个变成了必须的

**响应**

给个例子就好了

```properties
HTTP/1.1 302 Found
Location: https://client.example.org/cb#
  access_token=SlAV32hkKG
  &token_type=bearer
  &id_token=eyJ0 ... NiJ9.eyJ1c ... I6IjIifX0.DeWt4Qu ... ZXso
  &expires_in=3600
  &state=af0ifjsldkj
```

> 与前面不同的是，这里会多一个Access Token的验证，它要结合ID Token中的at_hash字段验证
>
> - 使用ID Token中头部alg字段指定的算法对Access Token进行哈希
> - 对哈希值进行base64url编码，取左半边
> - 上一步得到的值必须和ID Token中的at_hash字段指定的值匹配

这里需要说明的是：*OIDC*的说明文档里很明确的说明了用户的相关信息都要使用**JWT**形式编码。在*JWT*中，不应该在载荷里面加入任何敏感的数据。如果传输的是用户的User ID。这个值实际上不是什么敏感内容，一般情况下被知道也是安全的。

> 但是现在工业界已经**不推荐**使用*OAuth*默认模式，而推荐使用不带*client_Secret*的*授权码模式*。

### 混合模式

混合模式简而言之就是以上提到的两种模式的混合，不过也有一些小的改变，就是允许直接向客户端返回*Access-Token*。

业界普遍认为，后端传递Token（比如服务器之间通信）要比前端（比如页面之间）可靠，所以如果直接返回令牌的情况下会把令牌的过期时间设置较短。

1. RP准备请求
2. RP发送请求给OP
3. OP对用户鉴权
4. OP采集用户的鉴权和授权信息
5. OP发送给RP授权码，同时根据请求时指定的response_type，发送额外的参数
6. RP通过授权码向OP请求
7. RP从OP处得到ID Token和Access Token
8. RP验证ID Token，解析用户的唯一标识

**授权码请求**

混合模式在response_type上做文章，允许的值包括：code id_token、code token、code id_token token。可以看出，在授权码请求这一步，可能会返回授权码、ID Token、Access Token。下面是一个返回授权码和ID Token的例子
```properties
HTTP/1.1 302 Found
Location: https://client.example.org/cb#
code=SplxlOBeZQQYbYS6WxSbIA
&id_token=eyJ0 ... NiJ9.eyJ1c ... I6IjIifX0.DeWt4Qu ... ZXso
&state=af0ifjsldkj
```

> 如果返回值有ID Token，它会包含授权码的签名，对应其c_hash字段，计算方式和隐藏模式的Access Token验证差不多。

至此，混合模式看起来很奇怪，体现在两处

- ID Token既能在授权码请求时返回，也能在Access Token请求时返回
- Access Token也有上面的情况

### 小结

ID Token的意义，主要在于以安全的方式分发用户的唯一标识，Access Token才是用来作为访问凭证的。

### 获取用户信息

上面介绍了获取访问凭证之前的动作，这里介绍使用访问凭证访问ID Token指定的用户的信息的问题。

### 能获得什么

获得的内容和请求访问凭证时传的scope有关，具体如下。

![image-20221111200433193](SSO统一认证协议/image-20221111200433193.png)

> scope可以指定多个值，得到的结果就是并集。比如`scope=profile email phone address`

一个成功的例子

```properties
HTTP/1.1 200 OK
Content-Type: application/json

{
  "sub": "248289761001",
  "name": "Jane Doe",
  "given_name": "Jane",
  "family_name": "Doe",
  "preferred_username": "j.doe",
  "email": "janedoe@example.com",
  "picture": "http://example.com/janedoe/me.jpg"
}
```

### 另类获取信息的方式 - claims

可以在请求时加上claims参数，指定能够从用户信息端点或者在ID Token中包含什么信息，它是一个类似json schema的东西，如下是例子：要求给的字段，以及字段是否必要。

```properties
{
  "userinfo":
  {
    "given_name": {"essential": true},
    "nickname": null,
    "email": {"essential": true},
    "email_verified": {"essential": true},
    "picture": null,
    "http://example.info/claims/groups": null
  },
  "id_token":
  {
    "auth_time": {"essential": true},
    "acr": {"values": ["urn:mace:incommon:iap:silver"] }
  }
}
```

## 总结

理解了OAuth2.0的工作流程，再理解OpenID Connect就很容易了，相较而言，它的特点如下

- OP = 授权服务+资源服务，资源服务的唯一作用就是分发用户信息
- 规定了用户信息包含的内容
- Token响应中多了ID Token，它用来指定用户ID，以便在访问用户信息时作为标识
- ID Token使用了JWT
- 多了混合模式

# SAML

## 1. 什么是SAML协议？

SAML是Security Assertion Markup Language的简写，翻译过来叫安全断言标记语言，是一种基于XML的开源标准数据格式，用于在当事方之间交换身份验证和授权数据，尤其是在身份提供者（IDP）和服务提供者（SP）之间交换。SAML协议主要解决的是标准化、跨域、基于Web的单点登录（SSO）问题。SAML协议目前的版本是2.0，已经获得了行业的广泛认可，并被诸多主流厂商所支持，比如说我们经常使用的OKTA和G Suite等。

## 2. SAML协议的构成

SAML协议主要包含以下四方面内容：

- SAML Assertions 断言：定义交互的数据格式 (XML)

- SAML Protocols 协议：定义交互的消息格式 (XML+processing rules)

- SAML Bindings 绑定：定义如何与常见的通信协议绑定 (HTTP，SOAP)

- SAML Profile 使用框架：给出对 SAML 断言及协议如何使用的建议 (Protocols+Bindings)

  ![img](https:////upload-images.jianshu.io/upload_images/14307498-21ee5f1505a7392c.png?imageMogr2/auto-orient/strip|imageView2/2/w/974/format/webp)

  SAML协议的构成

## 3. SAML协议认证流程

常见的认证流程如下图所示。其中Service Provider为第三方服务，例如Timecard；User Agent为用户的浏览器；Identity Provider为认证服务器，例如OKTA。



![img](https:////upload-images.jianshu.io/upload_images/14307498-ace7f5f8d5a1dfc6.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1000/format/webp)

​																												SAML协议的认证流程图

1. 用户在浏览器中，试图访问第三方服务（SP）受保护的资源；

2. 如果用户没有登录，SP会初始化一个SAML请求（请求的示例参见下图），并请求浏览器使用重定向，将SAML请求转发到认证服务器（IdP），进行单点登录；

   ![img](https:////upload-images.jianshu.io/upload_images/14307498-aaf095ab69011cb8.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/998/format/webp)

   ​																								SAML认证请求

3. 浏览器接收到SP的SAML认证请求，向认证服务发起认证请求，用户使用IdP的账号密码完成登录；

4. 当IdP认证完成之后会返回用户的SAML断言信息（响应的示例参见下图），并请求浏览器通过重定向，将SAML断言结果重定向到SP的消费端点上；

   ![img](https:////upload-images.jianshu.io/upload_images/14307498-33e2f7935733e730.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

   ​																									SAML响应

5. 浏览器带上IdP返回的断言结果，请求SP的SAML消费端点，SP会完成SAML的安全校验，获取用户属性等，完成用户在SP上的登录；

6. 如果SAML断言标识用户已经认证通过，会再次请求浏览器重定向，将用户导向步骤1中请求的目标资源；

7. 浏览器向SP请求步骤1中的目标资源；

8. 用户已经登录认证成功，返回用户请求的目标资源。

或

![img](SSO统一认证协议/webp-1668234754867-68.webp)

1. 用户打开浏览器请求SP的受保护资源
2. SP收到请求后发现没有登录态，则生成saml request，同时请求IDP
3. IDP收到请求后，解析saml request(如果没有登录态) 然后重定向到登录页面
4. 用户在认证页面完成认证，再由IDP重定向到SP 的回调接口上
5. SP收到回掉信息后对response 解析校验，成功后生成SP侧的登录态

## 4. 服务构建

### IDP侧需要搭建SAML协议的服务端

常见的有：ADFS， AZURE 当然也有自研的Adfs， Azure 的搭建可以参考微软的官方文档

```properties
https://support.freshservice.com/support/solutions/articles/226938-configuring-adfs-for-freshservice-with-saml-2-0
```

### SAML request构造

SAMLRequest就是SAML认证请求消息。因为SAML是基于XML的比较长需要压缩和编码，在压缩和编码之前，SAMLRequest格式如下：

```xml
<samlp:AuthnRequest
AssertionConsumerServiceURL="https://www.xxx.cn/authentication/saml/idp/call_back"
Destination=""
ID="_c0c38877-6966-488f-8196-7dd6afd2a958"
IssueInstant="2020-11-17T03:18:46Z"
ProtocolBinding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
Version="2.0"
xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion"
xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol">
    <saml:Issuer>https://www.xxx.cn</saml:Issuer>
    <samlp:NameIDPolicy AllowCreate="true" Format=""/>
</samlp:AuthnRequest>
```

在这个xml中，我们需要填充3个信息：
 1: AssertionConsumerServiceURL(IDP 认证完成后的重定向地址)
 2: Destination(IDP 的目的地址)
 3:  saml:Issuer(请求方的信息) ；
 填充完成后再对xml进行Deflater编码 + Base64编码

填充xml的方法：

```java
public static String fillDestination(String destination) {
    SAXReader reader = new SAXReader();
    Document document = null;
    try {
        //document = reader.read(ResourceUtils.getFile("classpath:samlXml/samlRquestXml.xml"));
        //上面这种方式在idea上运行是可以的，但是打成jar包是会报文件找不到的异常
        ClassPathResource classPathResource = new ClassPathResource("samlXml/samlRquestXml.xml");
        document = reader.read(classPathResource.getInputStream());
        Element rootNode = document.getRootElement();
        List<Attribute> attributes = rootNode.attributes();
        for (Attribute a : attributes) {
            if (a.getName().equalsIgnoreCase("Destination")) {
                a.setValue(destination);
                break;
            }
        }

    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException | DocumentException e) {
        e.printStackTrace();
    }

    return document.asXML();
}
```

Deflater编码 + Base64编码

```java
private static String getString(String samlRequest) throws IOException {
    ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
    DeflaterOutputStream deflater = new DeflaterOutputStream(outputStream, new Deflater(Deflater.DEFLATED, true));
    try {
        deflater.write(samlRequest.getBytes(StandardCharsets.UTF_8));
        deflater.finish();
        // 2.Base64
        return Base64.encode(outputStream.toByteArray());
    } catch (Exception ex) {

    } finally {
        if (!ObjectUtils.isEmpty(outputStream)) {
            outputStream.close();
        }

        if (!ObjectUtils.isEmpty(deflater)) {
            deflater.close();
        }
    }

    return null;
}
```

注：Deflater默认是zlib压缩(会在xml上再加一层header) 同时，第2个参数要设置为true,因为这个字段为true则Deflater执行压缩的时候就不会把header信息序列化到xml 原文如下：

> If 'nowrap' is true then the ZLIB header and checksum fields will not be used in order to support the compression format used in both GZIP and PKZIP.

### 发送请求

```properties
https://adfs.xxxx.com/adfs/ls?SAMLRequest={第三步中生成的请求参数}
```

当IDP收到请求校验通过后就会重定向到自己的login页面，登录完成后再call back需要免登的应用服务

### Response解析

IDP验证完成后会生成response给我的应用服务，同时我们需要再对response进行相应的Base64 decode 和 inflater

```java
public static String xmlInflater(String samlRequest) throws IOException {

     byte[] decodedBytes = Base64.decode(samlRequest);

     StringBuilder stringBuffer = new StringBuilder();

     ByteArrayInputStream bytesIn = new ByteArrayInputStream(decodedBytes);

     InflaterInputStream inflater = new InflaterInputStream(bytesIn, new Inflater(true));

     try {

     byte[] b = new byte[1024];

     int len = 0;

     while (-1 != (len = inflater.read(b))) {

     stringBuffer.append(new String(b, 0, len));

     }

     } catch (Exception e) {

     //write log

     } finally {

     if (!ObjectUtils.isEmpty(inflater)) {

     inflater.close();;

     }

     if (!ObjectUtils.isEmpty(bytesIn)) {

     bytesIn.close();

     }

     }

     return stringBuffer.toString();

}
```

## 5. SAML协议的优点

- 支持跨域的SSO登录。
- 提升安全。对于用户而言，无需使用多套账号、密码，降低账号泄露的风险，对企业而言，可以集中管理员工的权限，提升了内部系统的安全；
- 改善用户体验，用户只需要提供一套账号、密码，就可以消费其它SP提供的第三方服务；
- 降低企业的管理成本；
- 支持多种SaaS应用；
- 无需新增代码，SP就能支持多租户，与多个IdP服务集成。

## 6. SAML协议的不足

- 不支持新的认证场景。SAML协议的整个流程依赖于浏览器的重定向，不支持device flow、委托授权等用户场景，OIDC可以作为一种成熟的替代解决方案。

## 7. 相关的工具推荐

- SAML tracer：Chrome的一个插件，用于解析和查看SAML的断言信息，可以让你更懂SAML。
- [SAML Tool](https://links.jianshu.com/go?to=https%3A%2F%2Fdevelopers.onelogin.com%2Fsaml%2Fonline-tools)：在线的SAML工具大全，你值得拥有。
- Python的SAML框架：Python3-SAML、SAML2。

## 8. 参考代码

- OneLogin的[Python3-SAML](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fonelogin%2Fpython3-saml)

## 9. 相关扩展

- [SAML协议文档](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.oasis-open.org%2Fcommittees%2Fdownload.php%2F11511%2Fsstc-saml-tech-overview-2.0-draft-03.pdf)
- [SAML协议的Wiki](https://links.jianshu.com/go?to=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FSecurity_Assertion_Markup_Language)

# 各协议简单对比

上面简单介绍了主流的几种SSO协议，本质上它们大同小异，都是基于中心信任的机制，服务提供者和身份提供者之间通过互信来交换用户信息，只是每个协议信息交换的细节不同，或者概念上有些不同。最后，通过一个简单对比表格来总结本文重点内容：

![img](SSO统一认证协议/webp-1668158399215-17.webp)
