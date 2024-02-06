## 刷新令牌策略

注意：刷新令牌只有在**授权码模式**和**密码模式**中才有，对应的指定这两种模式时，在类型上加上refresh_token

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

```java
/**
 * 当前需要使用内存方式存储了用户令牌，应当使用UserDetailsService才行，否则会报错
 */
@Component
public class MyUserDetailService implements UserDetailsService {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
        return new User("admin", passwordEncoder.encode("123456"),
                AuthorityUtils.commaSeparatedStringToAuthorityList("admin_role"));
    }
}
```

```java
@EnableWebSecurity
public class OAuth2SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private MyUserDetailService myUserDetailService;

    /**
     * password密码模式要使用此认证管理器
     */
    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    /**
     * 用户类信息
     */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(myUserDetailService);
    }
}
```

```java
@Configuration
@EnableAuthorizationServer
public class OAuth2AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private AuthenticationManager authenticationManager;
    
    @Autowired
    private MyUserDetailService myUserDetailService;

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                .withClient("test-pc")
                .secret(passwordEncoder.encode("123456"))
                .resourceIds("oauth2-server")
                .authorizedGrantTypes("password", "authorization_code", "refresh_token")
                .scopes("all")
                .autoApprove(false)
                .redirectUris("http://www.baidu.com/");
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        //密码模式需要配置认证管理器
        endpoints.authenticationManager(authenticationManager);
        //刷新令牌获取新令牌时需要
        endpoints.userDetailsService(myUserDetailService);
    }
}
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/12836966/1667724815614-ee3c6588-f57d-4c5f-a2e7-6c28f5bdd9ad.png#averageHue=%23f9f2f1&clientId=udf55d64e-4f3b-4&from=paste&height=220&id=u961b011c&originHeight=302&originWidth=858&originalType=binary&ratio=1&rotation=0&showTitle=false&size=16498&status=done&style=none&taskId=ub643d89a-08b2-4913-8f6f-9372ce136f3&title=&width=624)

![image.png](https://cdn.nlark.com/yuque/0/2022/png/12836966/1667724830385-215d5f19-1a23-47c4-814c-9a07aea63037.png#averageHue=%23f7f3f3&clientId=udf55d64e-4f3b-4&from=paste&height=233&id=ua20dabb5&originHeight=321&originWidth=748&originalType=binary&ratio=1&rotation=0&showTitle=false&size=30539&status=done&style=none&taskId=u1b62c713-edfa-4730-96d0-64852d68578&title=&width=544)

## 令牌管理策略

1. ResourceServerTokenServices接口定义了令牌加载、读取方法
2. AuthorizationServerTokenServices接口定义了令牌的创建、获取、刷新方法
3. ConsumerTokenServices定义了令牌的撤销方法(删除)
4. DefaultTokenServices实现了上述三个接口，它包含了一些令牌业务的实现，如创建令牌、读取令牌、刷新令牌、获取客户端ID。默认的创建一个令牌时，是使用 UUID 随机值进行填充的。除了持久化令牌是委托一个 **TokenStore** 接口实现以外，这个类几乎帮你做了所有事情

TokenStore接口负责持久化令牌，默认情况下，令牌是通过randomUUID产生的32位随机数来进行填充，从而产生的令牌默认是存储在内存中

- 内存存储采用的是TokenStore接口默认实现类InMemoryTokenStore，开发时方便调试，适用单机版
- RedisTokenStore将令牌存储到Redis非关系型数据库，适用于高并发服务
- JdbcTokenStore基于JDBC将令牌存储到关系型数据库中，可以在不同的服务器间共享令牌
- JWtTokenStore将用户信息存储到令牌中，这样后端就可以不存储，前端拿到令牌后可以直接解析出用户信息。

## 内存管理令牌

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
    
    @Bean
    public TokenStore redisTokenStore(){
		return new InMemoryTokenStore(); 
    }
    
    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }
}
```

```java
/**
 * 当前需要使用内存方式存储了用户令牌，应当使用UserDetailsService才行，否则会报错
 */
@Component
public class MyUserDetailService implements UserDetailsService {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
        return new User("admin", passwordEncoder.encode("123456"),
                AuthorityUtils.commaSeparatedStringToAuthorityList("admin_role"));
    }
}
```

```java
@EnableWebSecurity
public class OAuth2SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private MyUserDetailService myUserDetailService;

    /**
     * password密码模式要使用此认证管理器
     */
    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    /**
     * 用户类信息
     */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(myUserDetailService);
    }
}
```

```java
@Configuration
@EnableAuthorizationServer
public class OAuth2AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

    @Autowired
    private MyUserDetailsService myUserDetailsService;

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private PasswordEncoder passwordEncoder;
    
    @Autowired
    private TokenStore tokenStore;

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                .withClient("test-pc")
                .secret(passwordEncoder.encode("123456"))
                .resourceIds("oauth2-server")
                .authorizedGrantTypes("authorization_code", "password", "implicit", "client_credentials", "refresh_token")
                .scopes("all")
                .autoApprove(false)
                .redirectUris("http://www.baidu.com");
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints.authenticationManager(authenticationManager);
        endpoints.userDetailsService(myUserDetailsService);
        //令牌管理策略
        endpoints.tokenStore(tokenService());
    }
    
    @Bean 
    public AuthorizationServerTokenServices tokenService() { 
        DefaultTokenServices service=new DefaultTokenServices();
        //service.setClientDetailsService();//客户端详情服务
        service.setSupportRefreshToken(true);//支持刷新令牌
        service.setTokenStore(tokenStore);//令牌存储策略

        service.setAccessTokenValiditySeconds(7200); // 令牌默认有效期2小时
        service.setRefreshTokenValiditySeconds(259200); // 刷新令牌默认有效期3天
        return service;
    }
}
```

## Redis管理令牌

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
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
    
    @Autowired
    private RedisConnectionFactory redisConnectionFactory;
    
    @Bean
    public TokenStore redisTokenStore(){
        // redis管理令牌
        return new RedisTokenStore(redisConnectionFactory);
    }
    
    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }
}
```

```java
/**
 * 当前需要使用内存方式存储了用户令牌，应当使用UserDetailsService才行，否则会报错
 */
@Component
public class MyUserDetailService implements UserDetailsService {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
        return new User("admin", passwordEncoder.encode("123456"),
                AuthorityUtils.commaSeparatedStringToAuthorityList("admin_role"));
    }
}
```

```java
@EnableWebSecurity
public class OAuth2SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private MyUserDetailService myUserDetailService;

    /**
     * password密码模式要使用此认证管理器
     */
    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    /**
     * 用户类信息
     */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(myUserDetailService);
    }
}
```

```java
@Configuration
@EnableAuthorizationServer
public class OAuth2AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

    @Autowired
    private MyUserDetailsService myUserDetailsService;

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private PasswordEncoder passwordEncoder;
    
    @Autowired
    private TokenStore tokenStore;

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                .withClient("test-pc")
                .secret(passwordEncoder.encode("123456"))
                .resourceIds("oauth2-server")
                .authorizedGrantTypes("authorization_code", "password", "implicit", "client_credentials", "refresh_token")
                .scopes("all")
                .autoApprove(false)
                .redirectUris("http://www.baidu.com");
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints.authenticationManager(authenticationManager);
        endpoints.userDetailsService(myUserDetailsService);
        //令牌管理策略
        endpoints.tokenStore(tokenStore);
    }
}
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/12836966/1667726764653-4431b12a-1eaa-4e80-a03b-9a146752a8b4.png#averageHue=%231c1915&clientId=udf55d64e-4f3b-4&from=paste&height=150&id=ubbd8a01a&originHeight=206&originWidth=598&originalType=binary&ratio=1&rotation=0&showTitle=false&size=32081&status=done&style=none&taskId=u12037014-8b59-4324-aa3e-4fe1a8b25bc&title=&width=434.90909090909093)

## JDBC管理令牌

### 建表语句

具体SQL语句可以去官网查看

```sql
-- used in tests that use HSQL
create table oauth_client_details (
  client_id VARCHAR(128) PRIMARY KEY,
  resource_ids VARCHAR(256),
  client_secret VARCHAR(256),
  scope VARCHAR(256),
  authorized_grant_types VARCHAR(256),
  web_server_redirect_uri VARCHAR(256),
  authorities VARCHAR(256),
  access_token_validity INTEGER,
  refresh_token_validity INTEGER,
  additional_information VARCHAR(4096),
  autoapprove VARCHAR(256)
);
INSERT INTO `oauth_client_details` VALUES ('test-pc', 'oauth2-server,oauth2-resource', '$2a$10$Q2Dv45wFHgxQkFRaVNAzeOJorpTH2DwHb975VeHET30QsqwuoQOAe', 'all,Base_API', 'authorization_code,password,implicit,client_credentials,refresh_token', 'http://www.baidu.com/', NULL, 50000, NULL, NULL, 'false');

create table oauth_client_token (
  token_id VARCHAR(256),
  token BLOB,
  authentication_id VARCHAR(256) PRIMARY KEY,
  user_name VARCHAR(256),
  client_id VARCHAR(256)
);

create table oauth_access_token (
  token_id VARCHAR(256),
  token BLOB,
  authentication_id VARCHAR(256) PRIMARY KEY,
  user_name VARCHAR(256),
  client_id VARCHAR(256),
  authentication BLOB,
  refresh_token VARCHAR(256)
);

create table oauth_refresh_token (
  token_id VARCHAR(256),
  token BLOB,
  authentication BLOB
);

create table oauth_code (
  code VARCHAR(256), 
  authentication BLOB
);

create table oauth_approvals (
 userId VARCHAR(256),
 clientId VARCHAR(256),
 scope VARCHAR(256),
 status VARCHAR(10),
 expiresAt TIMESTAMP,
 lastModifiedAt TIMESTAMP
);


-- customized oauth_client_details table
create table ClientDetails (
  appId VARCHAR(256) PRIMARY KEY,
  resourceIds VARCHAR(256),
  appSecret VARCHAR(256),
  scope VARCHAR(256),
  grantTypes VARCHAR(256),
  redirectUrl VARCHAR(256),
  authorities VARCHAR(256),
  access_token_validity INTEGER,
  refresh_token_validity INTEGER,
  additionalInformation VARCHAR(4096),
  autoApproveScopes VARCHAR(256)
);
```

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.security.oauth</groupId>
    <artifactId>spring-security-oauth2</artifactId>
    <version>2.3.4.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.4.3.4</version>
</dependency>
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.2.8</version>
</dependency>
```

```yaml
server:
  port: 8080
spring:
  application:
    name: oauth2-server
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/oauth2?serverTimezone=Asia/Shanghai&useUnicode=true&characterEncoding=utf-8
    username: root
    password: 123456
    type: com.alibaba.druid.pool.DruidDataSource
```

```java
@Configuration
public class MyOauth2Config {

    /**
     * druid数据源
     */
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DataSource druidDataSource() {
        return new DruidDataSource();
    }

    /**
     * jdbc管理令牌
     */
    @Bean
    public TokenStore jdbcTokenStore() {
        return new JdbcTokenStore(druidDataSource());
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

```java
@Configuration
@EnableAuthorizationServer
public class OAuth2AuthenticationServerConfig extends AuthorizationServerConfigurerAdapter {

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private MyUserDetailsService myUserDetailsService;

    @Autowired
    private TokenStore tokenStore;

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                .withClient("test-pc")
                .secret(passwordEncoder.encode("123456"))
                .resourceIds("oauth2-server")
                .authorizedGrantTypes("authorization_code", "password", "implicit", "client_credentials", "refresh_token")
                .scopes("all")
                .autoApprove(false)
                .redirectUris("http://www.baidu.com");
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints.authenticationManager(authenticationManager);
        endpoints.userDetailsService(myUserDetailsService);
        //令牌管理策略
        endpoints.tokenStore(tokenStore);
    }
}
```

![image.png](https://cdn.nlark.com/yuque/0/2022/png/12836966/1667732810043-6278d964-0895-465e-a520-5f528c685ede.png#averageHue=%23f7f5f2&clientId=udf55d64e-4f3b-4&from=paste&height=52&id=udd5e426c&originHeight=72&originWidth=1369&originalType=binary&ratio=1&rotation=0&showTitle=false&size=9540&status=done&style=none&taskId=ube7f1c8c-ddfe-47f7-b2ea-fb0211569ed&title=&width=995.6363636363636)

## JWT管理令牌

```xml
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-jwt</artifactId>
    <version>1.1.1.RELEASE</version>
</dependency>
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
/**
 * 当前需要使用内存方式存储了用户令牌，应当使用UserDetailsService才行，否则会报错
 */
@Component
public class MyUserDetailService implements UserDetailsService {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
        return new User("admin", passwordEncoder.encode("123456"),
                AuthorityUtils.commaSeparatedStringToAuthorityList("admin_role"));
    }
}
```

### 实现TokenEnhancer自定义token内容增强器

```java
@Configuration
public class MyOAuth2Config {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public TokenStore tokenStore() {
        // JWT令牌存储方式
        return new JwtTokenStore(jwtAccessTokenConverter());
    }

    /**
     * 帮助JWT编码的令牌值在OAuth身份验证信息之间进行转换
     * JwtAccessTokenConverter是TokenEnhancer的一个实例
     */
    @Bean
    public JwtAccessTokenConverter jwtAccessTokenConverter() {
        JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
        // JWT签名的秘钥，这里使用的是对称加密，资源服务器使用该秘钥来验证
        converter.setSigningKey("jwt");
        return converter;
    }
}
```

```java
/**
 * token内容增强器
 */
@Component
public class JwtTokenEnhancer implements TokenEnhancer {
    
    @Override
    public OAuth2AccessToken enhance(OAuth2AccessToken oAuth2AccessToken, OAuth2Authentication oAuth2Authentication) {
        Map<String, Object> info = new HashMap<>();
        //为原有的token的载荷增加一些内容
        //在对token进行解密时就可以拿到这里添加的信息
        info.put("enhance", "enhance info");
        ((DefaultOAuth2AccessToken) oAuth2AccessToken).setAdditionalInformation(info);
        return oAuth2AccessToken;
    }
}
```

```java
@EnableWebSecurity
public class OAuth2SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private MyUserDetailService myUserDetailService;

    /**
     * password密码模式要使用此认证管理器
     */
    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    /**
     * 用户类信息
     */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(myUserDetailService);
    }
}
```

```java
@Configuration
@EnableAuthorizationServer
public class OAuth2AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private AuthenticationManager authenticationManager;

    @Autowired
    private MyUserDetailService myUserDetailService;
    
    @Autowired
    private TokenStore tokenStore;
    
    @Autowired
    private JwtAccessTokenConverter jwtAccessTokenConverter;

    @Autowired
    private JwtTokenEnhancer jwtTokenEnhancer;

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                .withClient("test-pc")
                .secret(passwordEncoder.encode("123456"))
                .resourceIds("oauth2-server")
                .authorizedGrantTypes("password", "authorization_code", "refresh_token")
                .scopes("all")
                .autoApprove(false)
                .redirectUris("http://www.baidu.com/");
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        //配置JWT的内容增强器，TokenEnhancer可以对token进行增强
        TokenEnhancerChain enhancerChain = new TokenEnhancerChain();
        List<TokenEnhancer> delegates = new ArrayList<>();
        //添加token增强器
        delegates.add(jwtTokenEnhancer);
        //添加转换器
        delegates.add(jwtAccessTokenConverter);
        //把增强内容放入增强链中
        enhancerChain.setTokenEnhancers(delegates);
        
        //密码模式需要配置认证管理器
        endpoints.authenticationManager(authenticationManager);
        //刷新令牌获取新令牌时需要
        endpoints.userDetailsService(myUserDetailService);
        endpoints.tokenStore(tokenStore);
        //配置JwtAccessToken转换器，将值转换为jwt
        endpoints.accessTokenConverter(jwtAccessTokenConverter);
        //配置token增强链
        endpoints.tokenEnhancer(enhancerChain);
    }

    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        //所有人都可访问/oauth/token_key，后面要获取公钥，默认拒绝访问
        security.tokenKeyAccess("permitAll()");
        //认证后可访问/oauth/check_token，默认拒绝访问
        security.checkTokenAccess("permitAll()");
    }
}
```

### 利用令牌管理服务管理JWT令牌

```java
@Configuration
public class MyOAuth2Config {
    
    @Autowired
    private JwtTokenEnhancer jwtTokenEnhancer;
    
    @Bean
    public PasswordEncoder passwordEncoder(){
        return new BCryptPasswordEncoder();
    }
    
    @Bean
    public TokenStore tokenStore(){
        return new JwtTokenStore(jwtAccessTokenConverter());
    }
    
    @Bean
    public JwtAccessTokenConverter jwtAccessTokenConverter(){
        JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
        converter.setSigningKey("123");
        return converter;
    }

    /**
     * 令牌管理服务
     */
    @Bean
    public AuthorizationServerTokenServices authorizationServerTokenServices(){
        DefaultTokenServices tokenServices = new DefaultTokenServices();
        // 客户端详情，因为是向客户端颁发令牌，所以需要知道是哪一个客户端
        /*tokenServices.setClientDetailsService();*/
        // 是否支持刷新令牌
        tokenServices.setSupportRefreshToken(true);
        // 令牌存储策略
        tokenServices.setTokenStore(tokenStore());

        // 设置令牌增强
        TokenEnhancerChain tokenEnhancerChain = new TokenEnhancerChain();
        List<TokenEnhancer> delegates = new ArrayList<>();
        delegates.add(jwtTokenEnhancer);
        delegates.add(jwtAccessTokenConverter());
        tokenEnhancerChain.setTokenEnhancers(delegates);
        tokenServices.setTokenEnhancer(tokenEnhancerChain);

        // access_token默认有效期2小时
        tokenServices.setAccessTokenValiditySeconds(7200);
        // refresh_token默认有效期3天
        tokenServices.setRefreshTokenValiditySeconds(259200);
        return tokenServices;
    }
}
```

```java
@Configuration
@EnableAuthorizationServer
public class OAuth2AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

    @Autowired
    private PasswordEncoder passwordEncoder;
    
    @Autowired
    private AuthenticationManager authenticationManager;
    
    @Autowired
    private MyUserDetailService myUserDetailService;
    
    @Autowired
    private AuthorizationServerTokenServices authorizationServerTokenServices;

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                .withClient("test-pc")
                .secret(passwordEncoder.encode("123456"))
                .resourceIds("oauth2-server")
                .authorizedGrantTypes("password", "authorization_code", "refresh_token")
                .scopes("all")
                .autoApprove(false)
                .redirectUris("http://www.baidu.com/");
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        //密码模式需要配置认证管理器
        endpoints.authenticationManager(authenticationManager);
        //刷新令牌获取新令牌时需要
        endpoints.userDetailsService(myUserDetailService);
        //令牌管理服务
        endpoints.tokenServices(authorizationServerTokenServices);
    }

    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        //所有人都可访问/oauth/token_key，后面要获取公钥，默认拒绝访问
        security.tokenKeyAccess("permitAll()");
        //认证后可访问/oauth/check_token，默认拒绝访问
        security.checkTokenAccess("permitAll()");
    }
}
```