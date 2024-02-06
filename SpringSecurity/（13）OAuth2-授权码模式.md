## 工作流程

![image-20240124211447098](C:\Users\pgx\AppData\Roaming\Typora\typora-user-images\image-20240124211447098.png)

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
                .authorizedGrantTypes("authorization_code","refresh_token")
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

**Authorization Request（获取code码）**

客户端通过使用“application/x-www-form- urlencoding”格式向授权端点URI的查询组件添加以下参数来构造请求URI

- response_type：必须的。值必须是"code"。
- client_id：必须的。客户端标识符。
- redirect_uri：可选的。
- scope：可选的。请求访问的范围。
- state：推荐的。一个不透明的值用于维护请求和回调之间的状态。授权服务器在将用户代理重定向会客户端的时候会带上该参数。

例如：http://localhost:8080/oauth/authorize?response_type=code&client_id=test-pc

**Authorization Response**

如果资源所有者授权访问请求，授权服务器发出授权代码并通过使用“application/x-www-form- urlencoding”格式向重定向URI的查询组件添加以下参数，将其给客户端。

- code：必须的。授权服务器生成的授权码。授权代码必须在发布后不久过期，以减少泄漏的风险。建议最大授权代码生命期为10分钟。客户端不得多次使用授权代码。如果授权代码不止一次使用，授权服务器必须拒绝请求，并在可能的情况下撤销先前基于该授权代码发布的所有令牌。授权代码是绑定到客户端标识符和重定向URI上的。
- state：如果之前客户端授权请求中带的有"state"参数，则响应的时候也会带上该参数。

例如：http://client.example.com/cb?code=SplxlOBeZQQYbYS6WxSbIA

**Access Token Request（用authorization_code模式获取token）**

客户端通过使用“application/ www-form-urlencoding”格式发送以下参数向令牌端点发出请求

- grant_type：必须的。值必须是"authorization_code"。
- code：必须的。值是从授权服务器那里接收的授权码。
- redirect_uri：如果在授权请求的时候包含"redirect_uri"参数，那么这里也需要包含"redirect_uri"参数。而且，这两处的"redirect_uri"必须完全相同。
- client_id：如果客户端不需要认证，那么必须带的该参数。

```txt
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded
　　grant_type=authorization_code&code=SplxlOBeZQQYbYS6WxSbIA&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb
```

例如：http://localhost:8080/oauth/token?client_id=test-pc&client_secret=123456&grant_type=authorization_code&code=MnPDgC

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
    "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
    "example_parameter":"example_value"
}
```

### Postman获取

**获取请求授权码code**

1. 使用该地址申请授权码：[http://localhost:8899/oauth/authorize?client_id=test-pc&response_type=code](http://localhost:8899/oauth/authorize?client_id=test-pc&response_type=code)

![image.png](https://cdn.nlark.com/yuque/0/2022/png/12836966/1667719776007-4f07a853-1310-4400-938f-9686498385d9.png#averageHue=%23f0f0f0&clientId=udf55d64e-4f3b-4&from=paste&height=308&id=u8b4887e4&originHeight=424&originWidth=949&originalType=binary&ratio=1&rotation=0&showTitle=false&size=14082&status=done&style=none&taskId=u2674591d-b57d-4a4c-8243-cd468dbb26b&title=&width=690.1818181818181)

- 此处输入的用户名、密码是在认证服务器输入的(看端口8899)，而不是在客户端上输入的，这样更加安全，因为客户端不知道用户名和密码
- 密码模式中，输入的用户名、密码不是在认证服务器上输入，而是在客户端输入的，这样客户端就不太安全。

![image.png](https://cdn.nlark.com/yuque/0/2022/png/12836966/1667719786382-4b151aea-440c-4f14-9ab5-ea8cc03e200f.png#averageHue=%23f7f6f5&clientId=udf55d64e-4f3b-4&from=paste&height=214&id=oF4gz&originHeight=294&originWidth=836&originalType=binary&ratio=1&rotation=0&showTitle=false&size=22070&status=done&style=none&taskId=u5b713e33-7e19-40e7-9d74-8dddcadf394&title=&width=608)

2. 点击登录以后，会跳转到指定的redirect_uri，回调路径会携带一个授权码（code）

![image.png](https://cdn.nlark.com/yuque/0/2022/png/12836966/1667719803661-834ce670-d6e4-4739-b525-01ea0a467e1a.png#averageHue=%23fdf9f9&clientId=udf55d64e-4f3b-4&from=paste&height=276&id=ue6a2023a&originHeight=379&originWidth=1004&originalType=binary&ratio=1&rotation=0&showTitle=false&size=20833&status=done&style=none&taskId=u49ac89f3-f845-44e9-82f0-c9bd414e1bd&title=&width=730.1818181818181)

3. 获取到授权码（code）后，就可以通过它来获取访问令牌（access_token）

**通过授权码获取令牌token**

1. POST方式请求：[http://localhost:8899/oauth/token](http://localhost:8899/oauth/token)，设置认证模式

![image.png](https://cdn.nlark.com/yuque/0/2022/png/12836966/1667719918951-7ec30769-93b5-4a99-b294-4bac71095460.png#averageHue=%23f8efee&clientId=udf55d64e-4f3b-4&from=paste&height=228&id=SGSz8&originHeight=314&originWidth=721&originalType=binary&ratio=1&rotation=0&showTitle=false&size=21730&status=done&style=none&taskId=uedeed455-25c2-43ff-b30b-d14d7116b71&title=&width=524.3636363636364)

2. 请求体中指定授权方式和授权码

![image.png](https://cdn.nlark.com/yuque/0/2022/png/12836966/1667720049913-ffc8c0e5-a2c0-4554-bf26-42722a0a041d.png#averageHue=%23f7f3f2&clientId=udf55d64e-4f3b-4&from=paste&height=189&id=u9cb4bab2&originHeight=260&originWidth=881&originalType=binary&ratio=1&rotation=0&showTitle=false&size=28317&status=done&style=none&taskId=uf2a30737-3ff4-4011-93ed-7f4c2dd8af8&title=&width=640.7272727272727)

3. 每个授权码申请令牌后就会失效，需要重新发送请求获取授权码再去认证，不然就会请求认证失败

![image.png](https://cdn.nlark.com/yuque/0/2022/png/12836966/1667720071225-ca992503-8580-46a4-bd46-3ba42625b21d.png#averageHue=%23fefefc&clientId=udf55d64e-4f3b-4&from=paste&height=122&id=ua1d7f24a&originHeight=168&originWidth=533&originalType=binary&ratio=1&rotation=0&showTitle=false&size=12694&status=done&style=none&taskId=ua0d04e75-7a98-4d04-85d9-0d56d7779c7&title=&width=387.6363636363636)

