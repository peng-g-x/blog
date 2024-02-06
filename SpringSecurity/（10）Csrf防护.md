## Csrf Token

从 Spring Security 4.x 开始，默认启用 CSRF 保护。该默认配置将 CSRF Token 添加到名为 _csrf 的 HttpServletRequest 属性中。

SpringSecurity的Csrf机制把请求方式分为两类来处理

1. GET、HEAD、TRACE、OPTIONS这四类请求可以直接通过
2. 除去上面，包括POST都要被验证携带token才能通过

为了保护 MVC 应用，Spring 会在每个生成的视图中添加一个 CSRF Token。该 Token 必须在每次修改状态的 HTTP 请求（PATCH、POST、PUT 和 DELETE）中提交给服务器。这可以保护应用免受 CSRF 攻击，因为攻击者无法从自己的页面获取此 Token。

用户登录时，系统发放一个CsrfToken值，用户携带该CsrfToken值与用户名、密码等参数完成登录，系统记录该会话的CsrfToken值，之后在用户的任何请求中，都必须带上该CsrfToken值，并由系统进行校验。这种方法需要与前端配置，包括存储CsrfToken值，以及在任何请求中（表单和ajax）携带CsrfToken值，这种配置会非常麻烦

_csrf 属性包含以下信息：

1. token：CSRF Token 值
2. parameterName：HTML 表单参数的名称，其中必须包含 Token 值
3. headerName：HTTP Header 的名称，其中必须包含 Token 值

**HTML表单**

如果视图使用 HTML 表单，可以使用 parameterName 和 token 值添加隐藏 input

```html
<input type="hidden" name="${_csrf.parameterName}" value="${_csrf.token}"/>
```

**JSON请求**

如果视图使用 JSON，则需要使用 headerName 和 token 值添加 HTTP 请求头信息。

1. 在 meta 标签中包含 Token 值和 Header 名称

```html
<meta name="_csrf" content="${_csrf.token}"/>
<meta name="_csrf_header" content="${_csrf.headerName}"/>
```

2. 用 JQuery 获取 meta 标签值

```js
var token = $("meta[name='_csrf']").attr("content");
var header = $("meta[name='_csrf_header']").attr("content");
```

3. 使用这些值来设置 XHR Header

```json
$(document).ajaxSend(function(e, xhr, options) {
    xhr.setRequestHeader(header, token);
});
```

## 无状态API

如果无状态 API 使用基于 Token 的身份验证（如 JWT），就不需要 CSRF 保护。反之，如果使用 Session Cookie 进行身份验证，就需要启用 CSRF 保护。无状态 API 无法像 MVC 配置那样添加CSRF Token，因为它不会生成任何 HTML 视图。

### 后端配置

```java
@Configuration
public class WebSecurityConfigurer extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf()
          .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse());
    }
}
```

在这种情况下，可以使用 CookieCsrfTokenRepository 在 Cookie 中发送 CSRF Token，此配置将为前端设置一个名为XSRF-TOKEN 的 Cookie。由于将 HTTP-only 标志设置为 false，因此前端能使用 JavaScript 获取此 Cookie。

### 前端配置

通过 JavaScript 从 document.cookie 列表中搜索 XSRF-TOKEN Cookie 值。

由于该列表以字符串形式存储，因此可以使用此 regex （正则）进行检索：

```js
const csrfToken = document.cookie.replace(/(?:(?:^|.*;\s*)XSRF-TOKEN\s*\=\s*([^;]*).*$)|^.*$/, '$1');
```

然后，必须向每个修改 API 状态的 REST 请求发送 Token： POST、PUT、DELETE 和 PATCH。Spring 会通过 X-XSRF-TOKEN Header 来接收它。只需使用 JavaScript Fetch API 设置即可：

```js
fetch(url, {
  method: 'POST',
  body: /* 发送给服务器的请求体 */,
  headers: { 'X-XSRF-TOKEN': csrfToken },
})
```

### JWT配置

```java
@Configuration
public class WebSecurityConfigurer extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable();
    }
}
```

## 工作原理

Spring Security 通过 CsrfFilter 实现 CSRF 防护。CsrfFilter 会直接放行 GET、HEAD、TRACE 和 OPTIONS 等请求，同时要求可能会修改数据的 PUT、POST 和 DELETE 等请求包含 CSRF Token 请求头或参数。如果 CSRF Token 不存在或值不正确，则拒绝该请求并将响应的状态设置为 403

### CsrfToken

```java
public interface CsrfToken extends Serializable {
	// 获取请求头名称
	String getHeaderName();
	// 获取应该包含 Token 的参数名称
	String getParameterName();
	// 获取具体的 Token 值
	String getToken();
}
```

### CsrfFilter

```java
@Override
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
    request.setAttribute(HttpServletResponse.class.getName(), response);
    // 从 CsrfTokenRepository 中获取当前用户的 CsrfToken
    CsrfToken csrfToken = this.tokenRepository.loadToken(request);
    boolean missingToken = (csrfToken == null);
    // 如果找不到 CsrfToken 就生成一个并保存到 CsrfTokenRepository 中
    if (missingToken) {
        csrfToken = this.tokenRepository.generateToken(request);
        this.tokenRepository.saveToken(csrfToken, request, response);
    }
    // 在请求中添加 CsrfToken
    request.setAttribute(CsrfToken.class.getName(), csrfToken);
    request.setAttribute(csrfToken.getParameterName(), csrfToken);
    // 如果是 "GET", "HEAD", "TRACE", "OPTIONS" 这些方法，直接放行
    if (!this.requireCsrfProtectionMatcher.matches(request)) {
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Did not protect against CSRF since request did not match "
                    + this.requireCsrfProtectionMatcher);
        }
        filterChain.doFilter(request, response);
        return;
    }
    // 从用户请求头中获取 CsrfToken
    String actualToken = request.getHeader(csrfToken.getHeaderName());
    if (actualToken == null) {
        // 头信息中拿不到，再从 param 中获取一次
        actualToken = request.getParameter(csrfToken.getParameterName());
    }
    // 如果请求所携带的 CsrfToken 与从 Repository 中获取的不同，则阻止访问
    if (!equalsConstantTime(csrfToken.getToken(), actualToken)) {
        this.logger.debug(
                LogMessage.of(() -> "Invalid CSRF token found for " + UrlUtils.buildFullRequestUrl(request)));
        AccessDeniedException exception = (!missingToken) ? new InvalidCsrfTokenException(csrfToken, actualToken): new MissingCsrfTokenException(actualToken);
        this.accessDeniedHandler.handle(request, response, exception);
        return;
    }
    // 正常情况下继续执行过滤器链的后续流程
    filterChain.doFilter(request, response);
}
```

### CsrfTokenRepository

```java
public interface CsrfTokenRepository {
	// 生成新的 token
	CsrfToken generateToken(HttpServletRequest request);
	// 保存 token，如果 token 传入 null 等同于删除
	void saveToken(CsrfToken token, HttpServletRequest request,
            HttpServletResponse response);
	// 从目标地点获取 token
	CsrfToken loadToken(HttpServletRequest request);
}
```

### CookieCsrfTokenRepository

它将 CsrfToken 值存储在用户的 cookie 内，减少了服务器 HttpSession 存储的内存消耗，并且当用 cookie 存储 CsrfToken 值时，前端可以用JS读取（需要设置该cookie的httpOnly属性为false），而不需要服务器注入参数。默认情况下 CookieCsrfTokenRepository 将编写一个名为 XSRF-TOKEN 的 cookie 和从头部命名 X-XSRF-TOKEN 或HTTP参数 _csrf 中读取

存储在cookie中是不可以被Csrf利用的，cookie只有在同域的情况下才能被读取，所以杜绝了第三方站点跨域读取CsrfToken值的可能。CSRF攻击本身是不知道cookie内容的，只是利用了当请求自动携带cookie时可以通过身份验证的漏洞，但服务器对CsrfToken值的校验并非取自cookie，而是需要前端手动将CsrfToken值作为参数携带在请求里

```java
@Override
public void saveToken(CsrfToken token, HttpServletRequest request, HttpServletResponse response) {
    // 判断参数 token 是否为空
    String tokenValue = (token != null) ? token.getToken() : "";
    // 根据 token，创建 Cookies
    Cookie cookie = new Cookie(this.cookieName, tokenValue);
    cookie.setSecure((this.secure != null) ? this.secure : request.isSecure());
    cookie.setPath(StringUtils.hasLength(this.cookiePath) ? this.cookiePath : this.getRequestContext(request));
    cookie.setMaxAge((token != null) ? this.cookieMaxAge : 0);
    cookie.setHttpOnly(this.cookieHttpOnly);
    if (StringUtils.hasLength(this.cookieDomain)) {
        cookie.setDomain(this.cookieDomain);
    }
    // 最终返回给浏览器
    response.addCookie(cookie);
}

@Override
public CsrfToken loadToken(HttpServletRequest request) {
    // 获取请求 Cookies
    Cookie cookie = WebUtils.getCookie(request, this.cookieName);
    if (cookie == null) {
        return null;
    }
    // 获取 Cookeis 中的 Token
    String token = cookie.getValue();
    if (!StringUtils.hasLength(token)) {
        return null; // 为空
    }
    // 获取到以后，创建 Token 对象
    return new DefaultCsrfToken(this.headerName, this.parameterName, token);
}
```

### HttpSessionCsrfTokenRepository

在默认情况下，SpringSecurity 加载的是一个HttpSessionCsrfTokenRepository，HttpSessionCsrfTokenRepository 将 CsrfToken 值存储在HttpSession中，并指定前端把 CsrfToken 值放在 "_csrf" 的请求参数或名为 "X-CSRF-TOKEN" 的请求头字段里。校验时，通过对比HttpSession内存储的CsrfToken值与前端携带的CsrfToken值是否一致，便能断定本次请求是否为CSRF攻击

```java
@Override
public void saveToken(CsrfToken token, HttpServletRequest request, HttpServletResponse response) {
    // 如果传入 token 为空，则删除当前会话的 Session
    if (token == null) {
        HttpSession session = request.getSession(false);
        if (session != null) {
            session.removeAttribute(this.sessionAttributeName);
        }
    }
    else {
        // 否则将 token 存入当前会话
        HttpSession session = request.getSession();
        session.setAttribute(this.sessionAttributeName, token);
    }
}
```

```java
@Override
public CsrfToken loadToken(HttpServletRequest request) {
    HttpSession session = request.getSession(false);
    if (session == null) {
        return null;
    }
    // 获取会话中的 Token 对象
    return (CsrfToken) session.getAttribute(this.sessionAttributeName);
}
```

### 自定义CsrfTokenRepository

```java
public interface JpaTokenRepository extends JpaRepository<Token, Integer> {
    Optional<Token> findTokenByIdentifier(String identifier);
}
```

```java
public final class DatabaseCsrfTokenRepository implements CsrfTokenRepository {

    private JpaTokenRepository jpaTokenRepository;
    @Autowired
    public void setJpaTokenRepository(JpaTokenRepository jpaTokenRepository) {
        this.jpaTokenRepository = jpaTokenRepository;
    }

	@Override
	public CsrfToken generateToken(HttpServletRequest request) {
		return new DefaultCsrfToken("X-XSRF-TOKEN", "_csrf", createNewToken());
	}

	@Override
	public void saveToken(CsrfToken token, HttpServletRequest request,
			HttpServletResponse response) {
		String identifier = request.getHeader("X-IDENTIFIER");
        Optional<Token> existingToken = jpaTokenRepository.findTokenByIdentifier(identifier);

        if (existingToken.isPresent()) {
            // token 已经存在，直接使用存在的
            Token token = existingToken.get();
            token.setToken(csrfToken.getToken());
        } else {
            // 否则创建 token，入库
            Token token = new Token();
            token.setToken(csrfToken.getToken());
            token.setIdentifier(identifier);
            jpaTokenRepository.save(token);
        }
    }

	@Override
	public CsrfToken loadToken(HttpServletRequest request) {
        String identifier = request.getHeader("X-IDENTIFIER");
        Optional<Token> existingToken = jpaTokenRepository.findTokenByIdentifier(identifier);

        if (existingToken.isPresent()) {
            // 查库查到了 token，则构造对象返回
            Token token = existingToken.get();
            return new DefaultCsrfToken("X-CSRF-TOKEN", "_csrf", token.getToken());
        }

        return null;
	}
}
```

```java
@Configuration
public class WebSecurityConfigurer extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().csrfTokenRepository(new CookieCsrfTokenRepository())
            	.and()
                .authorizeRequests()
                .anyRequest().authenticated()
                .and().formLogin()
                .defaultSuccessUrl("/") //登录认证成功后的跳转页面
                .and().httpBasic();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
        	.withUser("user").password("password").authorities("ROLE_USER")
        	.and().passwordEncoder(new PasswordEncoder() {
                @Override
                public String encode(CharSequence rawPassword) {
                    return rawPassword.toString();
                }

                @Override
                public boolean matches(CharSequence rawPassword, String encodedPassword) {
                    return rawPassword.equals(encodedPassword);
                }
        	});
    }
}
```