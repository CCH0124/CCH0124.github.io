---
title: Spring security 概觀
date: 2021-10-28
description: "Spring security"
tags: ["Spring security"]
draft: true
---

在 Spring Security 主要核心功能是 `Authentication`(使用者認證) 和 `Authorization`(使用者授權)兩部份。

1. Authentication
使用者是否能訪問該系統，一般都是透過帳號和密碼進行確認。
2. Authorization
用戶是否有權限執行某個操作，也就是系統上會有很多角色分配給使用者，而這些使用者能操作的動作就對應到角色。

## 演示
1. pom.xml
```xml
...
        <dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-security</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
...
```

2. 撰寫一個 controller

```java
@RestController
@RequestMapping("/api/v1")
public class TestController {
    @GetMapping("/hello")
    public String hello() {
        return "Hello Security";
    }
}
```

3. 對設計的 controller 進行請求
會發現會被導到一個登入頁面，這也可以表示 Spring Security 有被啟用。

![](https://i.imgur.com/f9qe0CR.png)
預設的使用者是 `user` 密碼則會是在 `console` 中以 `Using generated security password: d7ce1ace-a637-4896-8556-a531dedbab29` 呈現。登入後即可呈現我們所請求的內容。

## 基本原理
Spring Security 和 iptable 一樣都是一串鏈連接起來的。下面會介紹較為重要的過濾器

### FilterSecurityInterceptor
是一個方法級別的權限過濾器，基本上是鏈的最底部。
`FilterSecurityInterceptor` 實作了 `Filter`，其中包括以下方法

```java
    @Override
	public void init(FilterConfig arg0) {
	}

	/**
	 * Not used (we rely on IoC container lifecycle services instead)
	 */
	@Override
	public void destroy() {
	}

    @Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
			throws IOException, ServletException {
            invoke(new FilterInvocation(request, response, chain));
	}
```

在 `invoke` 方法中

```java
public void invoke(FilterInvocation filterInvocation) throws IOException, ServletException {
		if (isApplied(filterInvocation) && this.observeOncePerRequest) {
			// filter already applied to this request and user wants us to observe
			// once-per-request handling, so don't re-do security checking
			filterInvocation.getChain().doFilter(filterInvocation.getRequest(), filterInvocation.getResponse());
			return;
		}
		// first time this request being called, so perform security checking
		if (filterInvocation.getRequest() != null && this.observeOncePerRequest) {
			filterInvocation.getRequest().setAttribute(FILTER_APPLIED, Boolean.TRUE);
		}
		InterceptorStatusToken token = super.beforeInvocation(filterInvocation); // 前面的鏈必須通過才往下
		try {
			filterInvocation.getChain().doFilter(filterInvocation.getRequest(), filterInvocation.getResponse());
		}
		finally {
			super.finallyInvocation(token);
		}
		super.afterInvocation(token, null);
	}
```

### ExceptionTranslationFilter
顧名思義就是一個異常過濾器，用來處理認證授權過程中的異常。

### UsernamePasswordAuthenticationFilter
針對 `/login` 進行攔截請求，並檢驗起內容。

```java
    @Override
	public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
			throws AuthenticationException {
		if (this.postOnly && !request.getMethod().equals("POST")) { // 是否為 POST 請求
			throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
		}
		String username = obtainUsername(request); // 獲取使用者名稱
		username = (username != null) ? username : "";
		username = username.trim();
		String password = obtainPassword(request); // 獲取使用者密碼
		password = (password != null) ? password : "";
        // 教驗
		UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(username, password);
		// Allow subclasses to set the "details" property
		setDetails(request, authRequest);
		return this.getAuthenticationManager().authenticate(authRequest);
	}
```

## 過濾器加載過程
使用 Spring Security 時需配置過濾器，而 Spring boot 會幫我們自動配置。
### DelegatingFilterProxy
```java
@Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {

		// Lazily initialize the delegate if necessary.
		Filter delegateToUse = this.delegate;
		if (delegateToUse == null) {
			synchronized (this.delegateMonitor) {
				delegateToUse = this.delegate;
				if (delegateToUse == null) {
					WebApplicationContext wac = findWebApplicationContext();
					if (wac == null) {
						throw new IllegalStateException("No WebApplicationContext found: " +
								"no ContextLoaderListener or DispatcherServlet registered?");
					}
					delegateToUse = initDelegate(wac); // 初始化
				}
				this.delegate = delegateToUse;
			}
		}

		// Let the delegate perform the actual doFilter operation.
		invokeDelegate(delegateToUse, request, response, filterChain);
	}
    protected Filter initDelegate(WebApplicationContext wac) throws ServletException {
		String targetBeanName = getTargetBeanName();
		Assert.state(targetBeanName != null, "No target bean name set");
		Filter delegate = wac.getBean(targetBeanName, Filter.class); // 獲取 FilterChainProxy Bean
		if (isTargetFilterLifecycle()) {
			delegate.init(getFilterConfig());
		}
		return delegate;
	}
```

## 重要的 interface
### UserDetailsService
在正常情況下使用者帳號與密碼都是從資料庫中獲取，因此必須藉由此接口來自定義邏輯控制認證邏輯。
正常會繼層 `UsernamePasswordAuthenticationFilter` 並覆寫三個方法
- attemptAuthentication
- successfulAuthentication
- unsuccessfulAuthentication
實現 `UserDetailsService`，並與資料庫交互，返回 Spring Security 原生 `User` 物件
### PasswordEncoder
用於數據加密，也就是使用者密碼。

## 設置使用者帳號與密碼
1. 透過環境配置
```
spring.security.user.name=test
spring.security.user.password=test
```
2. 使用配置類
```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        // TODO Auto-generated method stub
        BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();
        String password = passwordEncoder.encode("test");
        auth.inMemoryAuthentication().withUser("test").password(password).roles("admin");
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

}
```

3. 自定義實現類
- 建立使用  `UserDetailsService` 實現類
```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Autowired
    UserDetailsService userDetailsService;
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        // TODO Auto-generated method stub
        auth.userDetailsService(userDetailsService).passwordEncoder(passwordEncoder());
    }

    @Bean
    PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```
- 覆寫方法，返回 `User` 物件，該物件有使用者名稱和密碼甚至是權限
```java
@Service("userDetailsService") // 對應 SecurityConfig 類別的 UserDetailsService
public class CustomUserDetailsService implements UserDetailsService {
    // 根據使用者名稱做什麼事
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // TODO Auto-generated method stub
        List<GrantedAuthority> auths = AuthorityUtils.commaSeparatedStringToAuthorityList("role");
        return new User("mary", new BCryptPasswordEncoder().encode("123456"), auths);
    }
    
}
```

## 整合資料庫
1. 使用 Mybatis 
```java
@Mapper
public interface UserMapper {
    @Select("SELECT * FROM public.user WHERE username = #{username}")
    User getUserByUsername(String username);
}
```
2. 使用 postgresql
3. 使用 lombok
```java
@Data
public class User {
    private String id;
    private String username;
    private String password;
}
```
在 github 上可以查看詳細的代碼。但在開發時我們都會用自己的登入頁面，而不會使用 Spring security 提供的頁面。以下來進行自定義頁面的演示。

1. 實現相關配置
```java
    // SecurityConfig 進行配置
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // TODO Auto-generated method stub
        http.formLogin() // 自定義編寫登入頁面
            .loginPage("/login.html")
            .loginProcessingUrl("/user/login") // 登入後訪問路徑
            .defaultSuccessUrl("/api/v1/index") // 登入後跳轉路徑
            .permitAll()
            .and().authorizeRequests()
            .antMatchers("/", "/api/v1/hello", "/user/login").permitAll() // 不需要認證即可訪問
            .anyRequest().authenticated()
            .and().csrf().disable(); // 關閉 csrf 保護
    }
```

建立一個簡單的 html 登入頁面
```html
    <form action="/user/login" method="post">
        User: <input type="text" name="username"> <!-- 須為 username Spring Security 才能獲取-->
        <br>
        Password: <input type="password" name="password"> <!-- 須為 password Spring Security 才能獲取-->
        <br>
        <input type="submit" value="login">
    </form>
```

這邊設定好後，`/api/v1/hello` 可直接訪問，`/api/v1/index` 會被跳轉至 login 頁面，登入後即可存取。
## 基於角色或權限的訪問控制
其作法有以下四種
- hasAuthority
	- 當前的請求者具有指定的權限，例如該使用者有 Admin 權限
- hasAnyAuthority
- hasRole
- hasAnyRole

### hasAuthority
以下的 `.antMatchers("/api/v1/index").hasAuthority("admins")` 當前使用者，只有具有 admins 權限才能存取
```java
	@Override
    protected void configure(HttpSecurity http) throws Exception {
        // TODO Auto-generated method stub
        http.formLogin() // 自定義編寫登入頁面
            .loginPage("/login.html")
            .loginProcessingUrl("/user/login") // 登入後訪問路徑
            .defaultSuccessUrl("/api/v1/index") // 登入後跳轉路徑
            .permitAll()
            .and().authorizeRequests()
            .antMatchers("/", "/api/v1/hello", "/user/login").permitAll() // 不需要認證即可訪問
            .antMatchers("/api/v1/index").hasAuthority("admins")
            .anyRequest().authenticated()
            .and().csrf().disable(); // 關閉 csrf 保護
    }
```
我們透過 `UserDetailsService` 的 `User` 物件中給予權限

```java
List<GrantedAuthority> auths = AuthorityUtils.commaSeparatedStringToAuthorityList("admins");
```
假設不匹配時，將返回 403 HTTP 代碼。但此種方法不適合一個使用者有多個權限角色。

### hasAnyAuthority
當前請求者有任何提供的角色，也就是一個使用者被綁定多個角色時。使用方式如下

```java
// antMatchers("/api/v1/index").hasAuthority("admins,manager") 無法應用於多角色
antMatchers("/api/v1/index").hasAnyAuthority("admins,manager")
```
### hasRole
如果請求者具備給定角色就允許訪問，否則回應 403 HTTP 代碼。

```java
antMatchers("/api/v1/index").hasRole("sale") 
```

在底層下說明需要以 `ROLE_` 進行開頭
```java
	private static String hasRole(String role) {
		Assert.notNull(role, "role cannot be null");
		Assert.isTrue(!role.startsWith("ROLE_"),
				() -> "role should not start with 'ROLE_' since it is automatically inserted. Got '" + role + "'");
		return "hasRole('ROLE_" + role + "')";
	}
```
以 `sale` 來說我們需設置
```java
List<GrantedAuthority> auths = AuthorityUtils.commaSeparatedStringToAuthorityList("ROLE_sale");
```
### hasAnyRole
表示能設定多個角色，只要符合一個即可存取。
```java
antMatchers("/api/v1/index").hasAnyRole("sale, pm") 
```

以上的方式只要沒有權限就回傳 403，我們可以自定義此頁面，並在 `configure` 方法進行配置即可
```java
	@Override
    protected void configure(HttpSecurity http) throws Exception {
        // TODO Auto-generated method stub
        http.exceptionHandling().accessDeniedPage("/403.html"); // 沒權限時自動跳轉此頁面
        http.formLogin() // 自定義編寫登入頁面
            ...
            .anyRequest().authenticated()
            .and().csrf().disable(); // 關閉 csrf 保護
    }
```


以上我們可以了解到基本的 Spring Security 配置以及權限方面控管。
