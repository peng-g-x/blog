## 工作流程

![image-20240124211531575](C:\Users\pgx\AppData\Roaming\Typora\typora-user-images\image-20240124211531575.png)

## 基本使用

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
    <version>2.3.12.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.security.oauth</groupId>
    <artifactId>spring-security-oauth2</artifactId>
    <version>2.3.4.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>2.3.12.RELEASE</version>
</dependency>
```

```java
@Configuration
public class MyOAuth2Config {

    /**
    * 加密方式
    */
    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }
}
```

1. 创建安全配置类：指定认证用户的用户名和密码，用户和密码是资源的所有者，
2. 创建认证服务器：这个客户端id和密码跟上面的用户名和密码是不一样的，客户端id和密码是应用系统的标识，每个应用系统对应一个客户端id和密码

```java
/**
 * 安全配置类
 */
@EnableWebSecurity
public class OAuth2SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private PasswordEncoder passwordEncoder;

    /**
     * 用户类信息
     */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
                .withUser("admin")
                .password(passwordEncoder.encode("123456"))
                .authorities("admin_roles");
    }
}
```

```java
/**
 * 认证服务器
 */
@Configuration
@EnableAuthorizationServer //开启认证服务器
public class OAuth2AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

    @Autowired
    private PasswordEncoder passwordEncoder;

    /**
     * 配置被允许访问此认证服务器的客户端详细信息
     * 1.内存管理
     * 2.数据库管理方式
     */
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                //客户端名称
                .withClient("test-pc")
                //客户端密码
                .secret(passwordEncoder.encode("123456"))
                //资源id，商品资源
                .resourceIds("oauth2-server")
                /**
                 * 授权类型，可同时支持多种授权类型
                 * authorization_code：授权码模式
                 * password：密码模式
                 * implicit：简化模式
                 * client_credentials：客户端模式
                 * refresh_token：更新令牌
                 */
                .authorizedGrantTypes("implicit","refresh_token")
                //授权范围标识，哪部分资源可访问（all是标识，不是代表所有）
                //比如指定微服务名称，则只可以访问指定的微服务
                .scopes("all")
                //false跳转到授权页面手动点击授权，true不用手动授权，直接响应授权码
                .autoApprove(false)
                //客户端回调地址，一定要和申请授权码时用的redirect_uri一致
                //当获取授权码后，认证服务器会重定向到指定的这个URL，并且带着一个授权码code响应
                .redirectUris("http://www.baidu.com/");
    }
}
```

## 获取Access Token

### 接口获取

**Authorization Request**

- response_type：必须的。值必须是"token"。
- client_id：必须的。
- redirect_uri：可选的。
- scope：可选的。

例如：

1. http://localhost:8080/oauth/authorize?response_type=token&client_id=banana&redirect_uri=http://baidu.com
2. http://localhost:8080/oauth/authorize?response_type=token&client_id=client1

### 浏览器获取

1. 输入访问地址：[http://localhost:8080/oauth/authorize?client_id=test-pc&response_type=token](http://localhost:8080/oauth/authorize?client_id=test-pc&response_type=token)
2. 当请求到达认证服务器的AuthorizationEndpoint后，它会要求资源所有者做身份验证.

![image.png](https://cdn.nlark.com/yuque/0/2022/png/12836966/1667722430725-53c636e9-6f97-44c9-a70d-95b19080bddd.png#averageHue=%23f1f1f0&clientId=udf55d64e-4f3b-4&from=paste&height=267&id=aR5AO&originHeight=367&originWidth=1007&originalType=binary&ratio=1&rotation=0&showTitle=false&size=13851&status=done&style=none&taskId=uef10f5f6-6549-4539-85f8-0307db4969c&title=&width=732.3636363636364)

3. 点击登录后，会跳转到指定的redirect_uri，回调路径会携带者令牌access_token、expires_in、scope等

![image.png](https://cdn.nlark.com/yuque/0/2022/png/12836966/1667722260228-b9f620c6-0c54-4b79-ad5a-2f77b7e86d46.png#averageHue=%23eff3d2&clientId=udf55d64e-4f3b-4&from=paste&height=95&id=u9e0922ac&originHeight=131&originWidth=1201&originalType=binary&ratio=1&rotation=0&showTitle=false&size=20001&status=done&style=none&taskId=u7a4fcae7-3fac-42b4-8484-713d74a3a1d&title=&width=873.4545454545455)
