---
title: Spring security - annotation
date: 2021-10-29
description: "Spring security 註解使用"
tags: ["Spring security"]
draft: true
---

註解是為了可以方便讓我們使用，以下將會介紹註解。實驗過程可藉由更改 `commaSeparatedStringToAuthorityList` 方法所傳遞的權限進行調整
```java
 List<GrantedAuthority> auths = AuthorityUtils.commaSeparatedStringToAuthorityList("admins,ROLE_sale");
```

### @Secured
判斷是否具有角色，有的話表示可以存取該方法，這邊匹配的角色需加上 **ROLE_**。在使用註解時須先開啟註解功能 `@EnableGlobalMethodSecurity`

```java
@SpringBootApplication
@EnableGlobalMethodSecurity(securedEnabled = true)
public class Securitydemo01Application {

	public static void main(String[] args) {
		SpringApplication.run(Securitydemo01Application.class, args);
	}

}
```
透過 `@Secured` 註解可讓該方法被限制只能是 "ROLE_sale" 或 "ROLE_manager" 才能存取。
```java
    @GetMapping("/update")
    @Secured(value = {"ROLE_sale","ROLE_manager"})
    public String update() {
        return "Hello update";
    }
```

### @PreAuthorize
進入該方法前的權限驗證，可將登入使用者的 `roles/permissions` 參數傳到方法中。同樣的要使用該註解必須啟用，在 `@EnableGlobalMethodSecurity` 添加 `prePostEnable=true`。

```java
    @GetMapping("/preAuth")
    @PreAuthorize(value = "hasAnyAuthority('admins')")
    public String preAuth() {
        return "Hello preAuth";
    }
```

其中 `hasAnyAuthority` 也可以是 `hasAuthority`、`hasRole` 或 `hasAnyRole` 等，以下整理了一張表

|Expression|Descript|
|---|---|
|hasRole([role])|當前請求者是否有指定角色|
|hasAnyRole([role1, role2])|當前請求者是否有符合設定的任一角色|
|hasAuthority([auth])|當前請求者是否有指定角色|
|hasAnyAuthority([auth1,auth2])|當前請求者是否有符合設定的任一角色|
|Principle|當前請求者 Principle 物件|
|authentication|從 SecurityContext 獲取當前 authentication|
|permitAll|都允許|
|dentAll|都拒絕|
|isAnonymous()|當前請求者是否為一個匿名使用者|
|isRememberMe()|表示當前請求者是否使用 Remember-Me 自動登入|
|isAuthenticated()|當前使用者是否登入認證成功|
|isFullyAuthenticated()|當前使用者非匿名且不透過 Remember-Me 登入|

### @PostAuthorize
在方法執行之後驗證權限。適合驗證帶有返回值的權限。同樣的要使用該註解必須啟用，在 `@EnableGlobalMethodSecurity` 添加 `prePostEnable=true`

```java
    @GetMapping("/postAuth")
    @PostAuthorize(value = "hasAnyAuthority('admins')")
    public String postAuth() {
        System.out.println("update...");
        return "Hello postAuth";
    }
```
同時調整 `List<GrantedAuthority> auths = AuthorityUtils.commaSeparatedStringToAuthorityList("ROLE_sale");`讓其只有 `ROLE_sale` 權限。會發現說方法被執行了，但頁面是 403 HTTP Code。

### @PostFilter
權限驗證之後對數據進行過濾。

下面範例 `filterObject` 獲取當前回傳值。
```java
    @GetMapping("/list")
    @PreAuthorize("hasRole('ROLE_manager')")
    @PostFilter("filterObject.username == 'admin1'")
    public List<User> getUserList() {
        List<User> list = new ArrayList<>();
        list.add(new User("1", "admin1", "password"));
        list.add(new User("2", "admin2", "password"));

        return list;
    }
```

### @PreFilter
進入 Controller 時對數據進行過濾。與 `PostFilter` 類似。
```java
    @GetMapping("/prefilter/list")
    @PreAuthorize("hasRole('ROLE_manager')")
    @PreFilter("filterObject.username == 'admin1'")
    public List<User> getUserListForPreFilter(@RequestBody List<User> list) {
        list.forEach(System.out::println);
        return list;
    }
```