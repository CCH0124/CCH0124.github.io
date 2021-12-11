---
title: Spring security - user logout 與自動登入
date: 2021-10-30
description: "Spring security 使用者註銷與自動登入"
tags: ["Spring security"]
draft: true
---
## logout
以下是一個使用者登出後的配置，登出成功後頁面會跳轉到 `/api/v1/index`
```java
@Override
    protected void configure(HttpSecurity http) throws Exception {
        // TODO Auto-generated method stub
        // logout
        http.logout().logoutUrl("/logout").logoutSuccessUrl("/api/v1/index").permitAll();
        http.exceptionHandling().accessDeniedPage("/403.html");
        http.formLogin() // 自定義編寫登入頁面
            ...
    }
```

撰寫一個登出頁面如下，同時也修改上面的 `defaultSuccessUrl` 配置將其換成 `defaultSuccessUrl("/success.html")`

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Success</title>
</head>
<body>
    <h1>Success</h1>
    <br>
    <a href="/logout"><b>logout</b></a>
</body>
</html>
```

實驗過程可以請求 `http://localhost:8080/login.html` 會跳轉到 `http://localhost:8080/success.html` 此時可以進行頁面的 API 請求。

## 自動登入
要實現的話會將數據儲存自 `cookie` 與 `database`，並進行比對。整體架構如下

```
 ________________                                                                _____________________________
|                |             _____________________________________            |                            |
|                |  1         |                                    |      2     |                            |
|                |----------->| UsernamePassworAuthenticationFilter|----------->|                            |
|                |            |____________________________________|            |                            |
|                |                                                              |    ___________________     |                  ________
|                |                                                              |    | RemeberMeService|     |                 |        |
|    browser     |                                                              |    |_________________|     |       4         |        |
|                |                            3                                 |    ___________________     | --------------> |   DB   |
|                |<-------------------------------------------------------------|    |  TokenRepository|     |       13        |        |
|                |                                                              |    |_________________|     | --------------> |________|
|                |                                                              |                            |                 
|                |             _____________________________________            |                            |
|                |  11        |                                    |            |                            |                 ______________________
|                |            |                                    |     12     |                            |       14       |                     |
|                |----------->|   RememberMeAuthenticationFilter   |----------->|                            | -------------->|  UserDetailsService |
|________________|            |____________________________________|            |____________________________|                |_____________________|
```
初始請求
1. 認證請求
2. 認證成功，調用 `successfulAuthentication`
3. 將 Token 寫入瀏覽起的 cookie
4. 將 Token 寫入 DB
在發送請求
11. 服務請求
12. 讀取 cookie 中的 Token
13. 查找 Token

在 `UsernamePasswordAuthenticationFilter` 所繼層的抽象類 `AbstractAuthenticationProcessingFilter` 中有一個 `successfulAuthentication` 方法，裡面有一個 `rememberMeServices` 它調用了 `loginSuccess`
```java
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain,
			Authentication authResult) throws IOException, ServletException {
		SecurityContextHolder.getContext().setAuthentication(authResult); // 獲取當前用戶權限
		if (this.logger.isDebugEnabled()) {
			this.logger.debug(LogMessage.format("Set SecurityContextHolder to %s", authResult));
		}
		this.rememberMeServices.loginSuccess(request, response, authResult);
		if (this.eventPublisher != null) {
			this.eventPublisher.publishEvent(new InteractiveAuthenticationSuccessEvent(authResult, this.getClass()));
		}
		this.successHandler.onAuthenticationSuccess(request, response, authResult);
	}
```
而 `loginSuccess` 實現如下
```java
    @Override
	public final void loginSuccess(HttpServletRequest request, HttpServletResponse response,
			Authentication successfulAuthentication) {
		if (!rememberMeRequested(request, this.parameter)) {
			this.logger.debug("Remember-me login not requested.");
			return;
		}
		onLoginSuccess(request, response, successfulAuthentication);
	}
```

當中在查看 `onLoginSuccess` 實現，

```java
   // PersistentTokenBasedRememberMeServices class
    @Override
	protected void onLoginSuccess(HttpServletRequest request, HttpServletResponse response,
			Authentication successfulAuthentication) {
		String username = successfulAuthentication.getName();
		this.logger.debug(LogMessage.format("Creating new persistent login for user %s", username));
		PersistentRememberMeToken persistentToken = new PersistentRememberMeToken(username, generateSeriesData(),
				generateTokenData(), new Date());
		try {
			this.tokenRepository.createNewToken(persistentToken); // 建立 Token
			addCookie(persistentToken, request, response); // 新增至 cookie
		}
		catch (Exception ex) {
			this.logger.error("Failed to save persistent token ", ex);
		}
	}

```

在往 `JdbcTokenRepositoryImpl` 追，可以知道說 `createNewToken` 與資料庫的交互
```java
    @Override
	public void createNewToken(PersistentRememberMeToken token) {
		getJdbcTemplate().update(this.insertTokenSql, token.getUsername(), token.getSeries(), token.getTokenValue(),
				token.getDate());
	}
```

接著再次請求後會觸發 `RememberMeAuthenticationFilter`，其裡面 `doFilter` 方法它做了以下事情

```java
    private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
			throws IOException, ServletException {
		if (SecurityContextHolder.getContext().getAuthentication() != null) {
			this.logger.debug(LogMessage
					.of(() -> "SecurityContextHolder not populated with remember-me token, as it already contained: '"
							+ SecurityContextHolder.getContext().getAuthentication() + "'"));
			chain.doFilter(request, response);
			return;
		}
		Authentication rememberMeAuth = this.rememberMeServices.autoLogin(request, response); // 自動登入，會從 cookie 拿值判別是否為空
		if (rememberMeAuth != null) {
			// Attempt authenticaton via AuthenticationManager
			try {
				rememberMeAuth = this.authenticationManager.authenticate(rememberMeAuth);
				// Store to SecurityContextHolder
				SecurityContextHolder.getContext().setAuthentication(rememberMeAuth);
				onSuccessfulAuthentication(request, response, rememberMeAuth);
				this.logger.debug(LogMessage.of(() -> "SecurityContextHolder populated with remember-me token: '"
						+ SecurityContextHolder.getContext().getAuthentication() + "'"));
				if (this.eventPublisher != null) {
					this.eventPublisher.publishEvent(new InteractiveAuthenticationSuccessEvent(
							SecurityContextHolder.getContext().getAuthentication(), this.getClass()));
				}
				if (this.successHandler != null) {
					this.successHandler.onAuthenticationSuccess(request, response, rememberMeAuth);
					return;
				}
			}
			catch (AuthenticationException ex) {
				this.logger.debug(LogMessage
						.format("SecurityContextHolder not populated with remember-me token, as AuthenticationManager "
								+ "rejected Authentication returned by RememberMeServices: '%s'; "
								+ "invalidating remember-me token", rememberMeAuth),
						ex);
				this.rememberMeServices.loginFail(request, response);
				onUnsuccessfulAuthentication(request, response, ex);
			}
		}
		chain.doFilter(request, response);
	}
```
`autoLogin` 邏輯
```java
    @Override
	public final Authentication autoLogin(HttpServletRequest request, HttpServletResponse response) {
		String rememberMeCookie = extractRememberMeCookie(request);
		if (rememberMeCookie == null) {
			return null;
		}
		this.logger.debug("Remember-me cookie detected");
		if (rememberMeCookie.length() == 0) {
			this.logger.debug("Cookie was empty");
			cancelCookie(request, response);
			return null;
		}
		try {
			String[] cookieTokens = decodeCookie(rememberMeCookie); // 解密
			UserDetails user = processAutoLoginCookie(cookieTokens, request, response);
			this.userDetailsChecker.check(user); // 與 DB 進行比對
			this.logger.debug("Remember-me cookie accepted");
			return createSuccessfulAuthentication(request, user);
		}
		catch (CookieTheftException ex) {
			cancelCookie(request, response);
			throw ex;
		}
		catch (UsernameNotFoundException ex) {
			this.logger.debug("Remember-me login was valid but corresponding user not found.", ex);
		}
		catch (InvalidCookieException ex) {
			this.logger.debug("Invalid remember-me cookie: " + ex.getMessage());
		}
		catch (AccountStatusException ex) {
			this.logger.debug("Invalid UserDetails: " + ex.getMessage());
		}
		catch (RememberMeAuthenticationException ex) {
			this.logger.debug(ex.getMessage());
		}
		cancelCookie(request, response);
		return null;
	}
```

## 實作自動登入
建立一張 SQL 表
```sql
CREATE TABLE IF NOT EXISTS public.persistent_logins (
    username varchar(64) NOT NULL,
    series varchar(64) NOT NULL primary key,
    token varchar(64) NOT NULL,
    last_used timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

調整 `JdbcTokenRepositoryImpl` 的數據來源

```java
    @Autowired
    private DataSource dataSource;

    @Bean
    public PersistentTokenRepository persistentTokenRepository() {
        JdbcTokenRepositoryImpl jdbcTokenRepositoryImpl = new JdbcTokenRepositoryImpl();
        jdbcTokenRepositoryImpl.setDataSource(dataSource);
        return jdbcTokenRepositoryImpl;
    }
```

在 `configure` 新增以下以實現自動登入

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        ...
            .and().rememberMe().tokenRepository(persistentTokenRepository())
            .tokenValiditySeconds(60) // 設置有效時間
            .userDetailsService(userDetailsService)
        ...
    }
```

修正先前的 `login.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <form action="/user/login" method="post">
        User: <input type="text" name="username">
        <br>
        Password: <input type="password" name="password">
        <br>
        <input type="checkbox" name="remember-me">自動登入 <!-- remember-me 必須這樣設置-->
        <br>
        <input type="submit" value="login">
    </form>
</body>
</html>
```

##### 測試
先到 `login.html` 頁面，勾選自動登入，使用開發工具可發現在 `cookie` 中有了 `remember-me` 的東西

![](https://i.imgur.com/pChjFUJ.png)

同樣我們到資料庫中也可以發現其幫我們插入了一筆資料

```sql
cch=# \dt
               List of relations
 Schema |       Name        | Type  |  Owner
--------+-------------------+-------+----------
 public | persistent_logins | table | postgres
 public | user              | table | itachi
(2 rows)

cch=# select * from persistent_logins
cch-# ;
 username |          series          |          token           |        last_used
----------+--------------------------+--------------------------+-------------------------
 itachi   | pLVn0k8bFK3NoyU6aBvC+w== | SzdGzAWjl3Q1HbMIh7yn0w== | 2021-10-31 12:50:12.814
(1 row)
```