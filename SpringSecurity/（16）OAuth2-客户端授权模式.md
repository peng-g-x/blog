## 工作流程

![image-20240124211549194](C:\Users\pgx\AppData\Roaming\Typora\typora-user-images\image-20240124211549194.png)

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
                .authorizedGrantTypes("client_credentials","refresh_token")
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

**Access Token Request（客户端授权模式获取Token）**

- grant_type：必须的。值必须是"client_credentials"。
- scope：可选的。

```txt
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
```

例如：http://localhost:8080/oauth/token?client_id=test-pc&client_secret=123456&grant_type=client_credentials

**Access Token Response**

```txt
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
    "access_token":"2YotnFZFEjr1zCsicMWpAA",
    "token_type":"example",
    "expires_in":3600,
    "example_parameter":"example_value"
}
```

### Postman获取

![image.png](https://cdn.nlark.com/yuque/0/2022/png/12836966/1667723183352-06ccad97-5f1f-4b06-bc44-4d22acfc0781.png#averageHue=%23f9f1f1&clientId=udf55d64e-4f3b-4&from=paste&height=218&id=ue09da815&originHeight=300&originWidth=780&originalType=binary&ratio=1&rotation=0&showTitle=false&size=16433&status=done&style=none&taskId=u3de6254c-dcb2-4e85-aac8-5880b05fe2d&title=&width=567.2727272727273)![image.png](https://cdn.nlark.com/yuque/0/2022/png/12836966/1667723201459-a6af83d9-9ffe-4ff3-b258-902f53739515.png#averageHue=%23f9f4f4&clientId=udf55d64e-4f3b-4&from=paste&height=165&id=u0668adc8&originHeight=227&originWidth=863&originalType=binary&ratio=1&rotation=0&showTitle=false&size=22441&status=done&style=none&taskId=u546cf225-c3d4-4d2e-aef5-2b84e3d0167&title=&width=627.6363636363636)
