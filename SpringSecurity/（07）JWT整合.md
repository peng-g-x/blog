## 依赖配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
    <version>2.9.0</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.1</version>
</dependency>
<dependency>
    <groupId>com.github.axet</groupId>
    <artifactId>kaptcha</artifactId>
    <version>0.0.9</version>
</dependency>
<dependency>
    <groupId>cn.hutool</groupId>
    <artifactId>hutool-all</artifactId>
    <version>5.7.15</version>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
</dependency>
<dependency>
    <groupId>commons-codec</groupId>
    <artifactId>commons-codec</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

## 自定义全局返回结果

```java
@Data
public class Result implements Serializable {
    private int code;
    private String msg;
    private Object data;

    public static Result succ(Object data) {
        return succ(200, "操作成功", data);
    }

    public static Result fail(String msg) {
        return fail(400, msg, null);
    }

    public static Result succ (int code, String msg, Object data) {
        Result result = new Result();
        result.setCode(code);
        result.setMsg(msg);
        result.setData(data);
        return result;
    }

    public static Result fail (int code, String msg, Object data) {
        Result result = new Result();
        result.setCode(code);
        result.setMsg(msg);
        result.setData(data);
        return result;
    }
}
```

## JWT配置类

```yaml
jwt:
  header: Authorization
  expire: 604800  #7天，s为单位
  secret: 123456
```

```java
@Data
@Component
@ConfigurationProperties(prefix = "jwt")
public class JwtUtils {
    
    private long expire;
    private String secret;
    private String header;

    /**
     * 生成JWT
     * @param username
     * @return
     */
    public String generateToken(String username){
        Date nowDate = new Date();
        Date expireDate = new Date(nowDate.getTime() + 1000 * expire);
        
        return Jwts.builder()
                .setHeaderParam("typ","JWT")
                .setSubject(username)
                .setIssuedAt(nowDate)
                .setExpiration(expireDate)  //7天过期
                .signWith(SignatureAlgorithm.HS512,secret)
                .compact();
    }

    /**
     * 解析JWT
     * @param jwt
     * @return
     */
    public Claims getClaimsByToken(String jwt){
        try {
            return Jwts.parser()
                    .setSigningKey(secret)
                    .parseClaimsJws(jwt)
                    .getBody();
        }catch (Exception e){
            return null;
        }
    }

    /**
     * 判断JWT是否过期
     * @param claims
     * @return
     */
    public boolean isTokenExpired(Claims claims){
        return claims.getExpiration().before(new Date());
    }
}
```

## 自定义登录处理器

登录失败后，我们需要向前端发送错误信息，登录成功后，我们需要生成JWT，并将JWT返回给前端

```java
/**
 * 登录成功控制器
 */
@Component
public class LoginSuccessHandler implements AuthenticationSuccessHandler {

    @Autowired
    private JwtUtils jwtUtils;

    @Override
    public void onAuthenticationSuccess(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Authentication authentication) throws IOException {
        httpServletResponse.setContentType("application/json;charset=UTF-8");
        ServletOutputStream outputStream = httpServletResponse.getOutputStream();

        //生成JWT，并放置到请求头中
        String jwt = jwtUtils.generateToken(authentication.getName());
        httpServletResponse.setHeader(jwtUtils.getHeader(), jwt);

        Result result = Result.succ("SuccessLogin");
        outputStream.write(JSONUtil.toJsonStr(result).getBytes(StandardCharsets.UTF_8));
        outputStream.flush();
        outputStream.close();
    }
}
```

```java
/**
 * 登录失败控制器
 */
@Component
public class LoginFailureHandler implements AuthenticationFailureHandler {

    @Override
    public void onAuthenticationFailure(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, AuthenticationException e) throws IOException {
        httpServletResponse.setContentType("application/json;charset=UTF-8");
        ServletOutputStream outputStream = httpServletResponse.getOutputStream();

        String errorMessage = "用户名或密码错误";
        Result result;
        if (e instanceof CaptchaException) {
            errorMessage = "验证码错误";
            result = Result.fail(errorMessage);
        } else {
            result = Result.fail(errorMessage);
        }
        outputStream.write(JSONUtil.toJsonStr(result).getBytes(StandardCharsets.UTF_8));
        outputStream.flush();
        outputStream.close();
    }
}
```

## 自定义登出处理器

在用户退出登录时，我们需将原来的JWT置为空返给前端，这样前端会将空字符串覆盖之前的jwt，JWT是无状态化的，销毁JWT是做不到的，JWT生成之后，只有等JWT过期之后，才会失效。因此我们采取置空策略来清除浏览器中保存的JWT。同时我们还要将我们之前置入SecurityContext中的用户信息进行清除，这可以通过创建SecurityContextLogoutHandler对象，调用它的logout方法完成

```java
/**
 * 登出处理器
 */
@Component
public class JWTLogoutSuccessHandler implements LogoutSuccessHandler {
    
    @Autowired
    private JwtUtils jwtUtils;
    
    @Override
    public void onLogoutSuccess(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Authentication authentication) throws IOException, ServletException {
        if (authentication!=null){
            new SecurityContextLogoutHandler().logout(httpServletRequest, httpServletResponse, authentication);
        }
        httpServletResponse.setContentType("application/json;charset=UTF-8");
        ServletOutputStream outputStream = httpServletResponse.getOutputStream();
        
        httpServletResponse.setHeader(jwtUtils.getHeader(), "");

        Result result = Result.succ("SuccessLogout");
        
        outputStream.write(JSONUtil.toJsonStr(result).getBytes(StandardCharsets.UTF_8));
        outputStream.flush();
        outputStream.close();
    }
}
```

## 验证码配置

```java
public class CaptchaException extends AuthenticationException {
    
    public CaptchaException(String msg){
        super(msg);
    }
}
```

```java
/**
 * 验证码配置
 */
@Configuration
public class KaptchaConfig {
    
    @Bean
    public DefaultKaptcha producer(){
        Properties properties = new Properties();
        properties.put("kaptcha.border", "no");
        properties.put("kaptcha.textproducer.font.color", "black");
        properties.put("kaptcha.textproducer.char.space", "4");
        properties.put("kaptcha.image.height", "40");
        properties.put("kaptcha.image.width", "120");
        properties.put("kaptcha.textproducer.font.size", "30");
        Config config = new Config(properties);
        DefaultKaptcha defaultKaptcha = new DefaultKaptcha();
        defaultKaptcha.setConfig(config);
        return defaultKaptcha;
    }
}
```

```java
@RestController
public class CaptchaController {

    @Autowired
    private Producer producer;
    @Autowired
    private RedisUtil redisUtil;

    @GetMapping("/captcha")
    public void imageCode(HttpServletRequest request, HttpServletResponse response) throws IOException {
        String code = producer.createText();
        BufferedImage image = producer.createImage(code);

        redisUtil.set("captcha", code, 120);

        // 将验证码图片返回，禁止验证码图片缓存
        response.setHeader("Cache-Control", "no-store");
        response.setHeader("Pragma", "no-cache");
        response.setDateHeader("Expires", 0);
        response.setContentType("image/jpeg");
        ImageIO.write(image, "jpg", response.getOutputStream());
    }
}
```

## 自定义过滤器

OncePerRequestFilter：在每次请求时只执行一次过滤，保证一次请求只通过一次filter，而不需要重复执行

因为验证码是一次性使用的，一个验证码对应一个用户的一次登录过程，所以需用hdel将存储的key删除。当校验失败时，则交给登录认证失败处理器LoginFailureHandler进行处理

```java
/**
 * 验证码过滤器
 */
@Component
public class CaptchaFilter extends OncePerRequestFilter {

    @Autowired
    private RedisUtil redisUtil;
    @Autowired
    private LoginFailureHandler loginFailureHandler;

    @Override
    protected void doFilterInternal(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, FilterChain filterChain) throws ServletException, IOException {
        String url = httpServletRequest.getRequestURI();
        if ("/login/form".equals(url) && httpServletRequest.getMethod().equals("POST")) {
            //校验验证码
            try {
                validate(httpServletRequest);
            } catch (CaptchaException e) {
                //交给认证失败处理器
                loginFailureHandler.onAuthenticationFailure(httpServletRequest, httpServletResponse, e);
                return;
            }
        }
        filterChain.doFilter(httpServletRequest, httpServletResponse);
    }

    private void validate(HttpServletRequest request) {
        String code = request.getParameter("code");

        if (StringUtils.isBlank(code)) {
            throw new CaptchaException("验证码错误");
        }
        String captcha = (String) redisUtil.get("captcha");
        if (!code.equals(captcha)) {
            throw new CaptchaException("验证码错误");
        }

        //若验证码正确，执行以下语句，一次性使用
        redisUtil.del("captcha");
    }
}
```

**login.html**

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>登录</title>
</head>
<body>
<h3>表单登录</h3>
<form method="post" th:action="@{/login/form}">
    <input type="text" name="username" placeholder="用户名"><br>
    <input type="password" name="password" placeholder="密码"><br>
    <input type="text" name="code" placeholder="验证码"><br>
    <img th:onclick="this.src='/captcha?'+Math.random()" th:src="@{/captcha}" alt="验证码"/><br>
    <div th:if="${param.error}">
        <span th:text="${session.SPRING_SECURITY_LAST_EXCEPTION.message}" style="color:red">用户名或密码错误</span>
    </div>
    <button type="submit">登录</button>
</form>
</body>
</html>
```

BasicAuthenticationFilter：OncePerRequestFilter执行完后，由BasicAuthenticationFilter检测和处理**http basic认证**，取出请求头中的jwt，校验jwt

当前端发来的请求有JWT信息时，该过滤器将检验JWT是否正确以及是否过期，若检验成功，则获取JWT中的用户名信息，检索数据库获得用户实体类，并将用户信息告知Spring Security，后续我们就能调用security的接口获取到当前登录的用户信息。

若前端发的请求不含JWT，我们也不能拦截该请求，因为一般的项目都是允许匿名访问的，有的接口允许不登录就能访问，没有JWT也放行是安全的，因为我们可以通过Spring Security进行权限管理，设置一些接口需要权限才能访问，不允许匿名访问

```java
/**
 * JWT过滤器
 */
public class JwtAuthenticationFilter extends BasicAuthenticationFilter {

    @Autowired
    private JwtUtils jwtUtils;
    @Autowired
    private UserDetailServiceImpl userDetailService;

    public JwtAuthenticationFilter(AuthenticationManager authenticationManager) {
        super(authenticationManager);
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {
        String jwt = request.getHeader(jwtUtils.getHeader());
        //这里如果没有jwt，继续往后走，因为后面还有鉴权管理器等去判断是否拥有身份凭证，所以是可以放行的
        //没有jwt相当于匿名访问，若有一些接口是需要权限的，则不能访问这些接口
        if (StrUtil.isBlankOrUndefined(jwt)) {
            chain.doFilter(request, response);
            return;
        }

        Claims claim = jwtUtils.getClaimsByToken(jwt);
        if (claim == null) {
            throw new JwtException("token异常");
        }
        if (jwtUtils.isTokenExpired(claim)) {
            throw new JwtException("token已过期");
        }
        String username = claim.getSubject();
        User user = UserDetailServiceImpl.userMap.get(username);

        //构建token，这里密码为null，是因为提供了正确的JWT，实现自动登录
        UsernamePasswordAuthenticationToken token =
                new UsernamePasswordAuthenticationToken(username, null, userDetailService.getUserAuthority(user.getId()));
        SecurityContextHolder.getContext().setAuthentication(token);
        chain.doFilter(request, response);
    }
}
```

## 自定义权限异常处理器

当BasicAuthenticationFilter认证失败的时候会进入AuthenticationEntryPoint

我们之前放行了匿名请求，但有的接口是需要权限的，当用户权限不足时，会进入AccessDenieHandler进行处理

```java
/**
 * JWT认证失败处理器
 */
@Component
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {
    
    @Override
    public void commence(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, AuthenticationException e) throws IOException, ServletException {
        httpServletResponse.setContentType("application/json;charset=UTF-8");
        httpServletResponse.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        ServletOutputStream outputStream = httpServletResponse.getOutputStream();

        Result result = Result.fail("请先登录");
        outputStream.write(JSONUtil.toJsonStr(result).getBytes(StandardCharsets.UTF_8));
        outputStream.flush();
        outputStream.close();
    }
}
```

```java
/**
 * 无权限访问的处理
 */
@Component
public class JwtAccessDeniedHandler implements AccessDeniedHandler {
    
    @Override
    public void handle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, AccessDeniedException e) throws IOException, ServletException {
        httpServletResponse.setContentType("application/json;charset=UTF-8");
        httpServletResponse.setStatus(HttpServletResponse.SC_FORBIDDEN);
        ServletOutputStream outputStream = httpServletResponse.getOutputStream();

        Result result = Result.fail(e.getMessage());
        
        outputStream.write(JSONUtil.toJsonStr(result).getBytes(StandardCharsets.UTF_8));
        outputStream.flush();
        outputStream.close();
    }
}
```

## 自定义用户登录逻辑

```java
public class AccountUser implements UserDetails {

    private Long userId;

    private static final long serialVersionUID = 540L;
    private static final Log logger = LogFactory.getLog(User.class);
    private String password;
    private final String username;
    private final Collection<? extends GrantedAuthority> authorities;
    private final boolean accountNonExpired;
    private final boolean accountNonLocked;
    private final boolean credentialsNonExpired;
    private final boolean enabled;

    public AccountUser(Long userId, String username, String password, Collection<? extends GrantedAuthority> authorities) {
        this(userId, username, password, true, true, true, true, authorities);
    }

    public AccountUser(Long userId, String username, String password, boolean enabled, boolean accountNonExpired, boolean credentialsNonExpired, boolean accountNonLocked, Collection<? extends GrantedAuthority> authorities) {
        Assert.isTrue(username != null && !"".equals(username) && password != null, "Cannot pass null or empty values to constructor");
        this.userId = userId;
        this.username = username;
        this.password = password;
        this.enabled = enabled;
        this.accountNonExpired = accountNonExpired;
        this.credentialsNonExpired = credentialsNonExpired;
        this.accountNonLocked = accountNonLocked;
        this.authorities = authorities;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return this.authorities;
    }

    @Override
    public String getPassword() {
        return this.password;
    }

    @Override
    public String getUsername() {
        return this.username;
    }

    @Override
    public boolean isAccountNonExpired() {
        return this.accountNonExpired;
    }

    @Override
    public boolean isAccountNonLocked() {
        return this.accountNonLocked;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return this.credentialsNonExpired;
    }

    @Override
    public boolean isEnabled() {
        return this.enabled;
    }
}
```

```java
@Service
public class UserDetailServiceImpl implements UserDetailsService {

     public static Map<String, User> userMap = new HashMap<>();

    static {
        userMap.put("root", new User("root", "123", AuthorityUtils.createAuthorityList("all")));
    }

    @Resource
    private PasswordEncoder passwordEncoder;

    @Override
    public UserDetails loadUserByUsername(String s) throws UsernameNotFoundException {
        User user = userMap.get(s);
        if (user == null) {
            throw new UsernameNotFoundException("用户名或密码错误");
        }
        return new AccountUser(1L, user.getUsername(), passwordEncoder.encode(user.getPassword()), user.getAuthorities());
    }
}
```

```java
@Component
public class PasswordEncoder extends BCryptPasswordEncoder {

    /**
     * 加密
     * @param charSequence  明文字符串
     * @return
     */
    @Override
    public String encode(CharSequence charSequence) {
        try {
            MessageDigest digest = MessageDigest.getInstance("MD5");
            return toHexString(digest.digest(charSequence.toString().getBytes()));
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
            return "";
        }
    }

    /**
     * 密码校验
     * @param charSequence 明文，页面收集密码
     * @param s 密文 ，数据库中存放密码
     * @return
     */
    @Override
    public boolean matches(CharSequence charSequence, String s) {
        return s.equals(encode(charSequence));
    }

    /**
     * @param tmp 转16进制字节数组
     * @return 饭回16进制字符串
     */
    private String toHexString(byte [] tmp){
        StringBuilder builder = new StringBuilder();
        for (byte b :tmp){
            String s = Integer.toHexString(b & 0xFF);
            if (s.length()==1){
                builder.append("0");
            }
            builder.append(s);
        }
        return builder.toString();
    }
}
```

```java
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    LoginFailureHandler loginFailureHandler;
    @Autowired
    LoginSuccessHandler loginSuccessHandler;
    @Autowired
    CaptchaFilter captchaFilter;
    @Autowired
    JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;
    @Autowired
    JwtAccessDeniedHandler jwtAccessDeniedHandler;
    @Autowired
    UserDetailServiceImpl userDetailService;
    @Autowired
    JWTLogoutSuccessHandler jwtLogoutSuccessHandler;

    @Bean
    JwtAuthenticationFilter jwtAuthenticationFilter() throws Exception {
        JwtAuthenticationFilter jwtAuthenticationFilter = new JwtAuthenticationFilter(authenticationManager());
        return jwtAuthenticationFilter;
    }

    private static final String[] URL_WHITELIST = {
            "/login",
            "/logout",
            "/captcha",
            "/favicon.ico"
    };

    @Bean
    public PasswordEncoderImpl passwordEncoder() {
        return new PasswordEncoderImpl();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.cors()
                .and()
                .csrf().disable()
                //登录配置
                .formLogin()
                .loginPage("/login")
                .loginProcessingUrl("/login/form")
                .successHandler(loginSuccessHandler)
                .failureHandler(loginFailureHandler)

                .and()
                .logout()
                .logoutSuccessHandler(jwtLogoutSuccessHandler)

                //禁用session
                .and()
                .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)

                //配置拦截规则
                .and()
                .authorizeRequests()
                .antMatchers(URL_WHITELIST).permitAll()
                .anyRequest().authenticated()

                //异常处理器
                .and()
                .exceptionHandling()
                .authenticationEntryPoint(jwtAuthenticationEntryPoint)
                .accessDeniedHandler(jwtAccessDeniedHandler)

                //配置自定义的过滤器
                .and()
                .addFilter(jwtAuthenticationFilter())
                //验证码过滤器放在UsernamePassword过滤器之前
                .addFilterBefore(captchaFilter, UsernamePasswordAuthenticationFilter.class);
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailService)
                .passwordEncoder(passwordEncoder());
    }

    /**
     * 定制一些全局性的安全配置，例如：不拦截静态资源的访问
     */
    @Override
    public void configure(WebSecurity web) throws Exception {
        // 静态资源的访问不需要拦截，直接放行
        web.ignoring().antMatchers("/**/*.css", "/**/*.js", "/**/*.png", "/**/*.jpg", "/**/*.jpeg");
    }
}
```

