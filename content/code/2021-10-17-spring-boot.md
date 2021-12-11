---
title: Spring boot 概觀
date: 2021-10-17
description: "Spring boot2"
tags: ["Spring boot"]
draft: true
---

## Spring 能做什麼
- Microservice
- Reactive
- Cloud
- Web apps
- Serverless
    - FaaS
- Event Driven
- Batch

Spring 生態圈很大。包含以下
- Spring Framework
    - DI
    - AOP
- Spring Data
    - 數據操作部分
- Spring Cloud
- Spring Security
- Spring Session

Spring boot 底層是 Spring Framework。Spring5 後有了變化，有了 spring reactive
![](https://spring.io/images/diagram-reactive-1290533f3f01ec9c57baf2cc9ea9fa2f.svg)

JAVA 8 的特性也改變底層的實現原理，像是預設的 interface。

## 什麼是 Spring boot
Spring boot makes it easy to create stand-alone, production-grade Spring based Application that you can "just run".

其優點官方描述有以下
- Create stand-aline Spring application
- Embed Tomacat, Jetty or Undertow directly (no need to deploy WAR files)
- Provide opinionated 'starter' dependencies to simplify you build configuration
    - 自動 starter 依賴，簡化建構配置
- Automatically configure Sping and 3rd party libraries whenever possible
- Provide productioon-ready features such as metric, health checks, and externalized configuration
- Absoullutely no code gneration and no requirement for XML configuration

也因為封裝太好，內部原始碼複雜，不容易精通。

## 建立一個 Spring boot 專案
使用 Maven 啟動項目後其 pom.xml 如下配置

```xml= 
    <packaging>jar</packaging>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.5.5</version>
    </parent>
    
    <dependencies>
<!--     引入開發 Web 包     -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
<!--   將應用打包成 jar   -->
    <build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

建立主程序
```java=
@SpringBootApplication // 表示這是一個 Spring boot 應用
public class MainApplication {
    public static void main(String[] atgs) {
        SpringApplication.run(MainApplication.class, args);
    }
}
```

撰寫業務

```java=
@RestController
public class HelloController {
    @RequestMapping("/hello")
    public String handle01(){
        return "Hello";
    }
}
```
透過 application.properties 可進行環境配置像是端口等，否則都會是預設值。最後透過 maven 打包成 jar 檔即可運行。

## 自動配置原理
### 依賴管理
在 maven 配置中，使用了 parent 進行依賴管理
```xml=
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.5.5</version>
    </parent>
    而它的父項目是
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependncies</artifactId>
        <version>2.5.5</version>
    </parent>
    在往上追蹤可以看到其聲明了常用的開發依賴版本號，因此會自動賦予版本
```

官方提供了很多關於 starter 場景啟動器，可參考[官方](https://docs.spring.io/spring-boot/docs/current/reference/html/using.html#using.build-systems.starters)，只要引入，該場景的所有依賴都會自動引入。

而這些依賴幫我們配置了以下
- 自動配好 Tomcat
    - 在 `spring-boot-starter-web` 中自動引入 Tomcat
- 自動配至 SpringMVC
    - 與上同樣
    - 同時也幫我們做了編碼
- 默認 Package 結構
    - 主程序所在包及其下面的所有子包中組件都會被默認掃描進來
    - 無須先前的包掃描
    - 想要改變掃描包的路徑可以透過 `@SpringBootApplication(scanBasePackages="PACKAGE_PATH")` 或是 `@ComponentScan(PACKAGE_NAME)`
    
```java=
@SpringBootApplication 
相當於
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan("PACKAGE_NAME")
```
- 擁有默認值
    - 配置檔案(application.properties)的值終究會綁定到每個類上，這個類會在容器中創建對象
- 按需求加載自動配至資源
    - 依照引入的場景並自動配置該場景環境

## 容器功能
1. @Configuration
- 告訴 SpringBoot 這是一個配置類
- 透過 @Bean 給容器添加組件，以方法名作為組件的 ID。返回類型就是組件類型，返回的值，就是組件在容器中的實例

```java=
@Configuration
public class MyConfig {
    @Bean
    public User user01() {
        return User.builder().name("name").age(12).build();
    }

    @Bean()
    public Pet pet() {
        return Pet.builder().name("tomcat").build();
    }
}
```

```java=
@SpringBootApplication
public class DemoApplication {
    
	public static void main(String[] args) {
        // 返回 IOC 容器
		ConfigurableApplicationContext run =  SpringApplication.run(DemoApplication.class, args);
		String[] names = run.getBeanDefinitionNames();
        // 查看使否存在於 IOC 容器中
		Arrays.stream(names).filter(i -> Objects.equals("user01", i)).forEach(System.out::println);
		Arrays.stream(names).filter(i -> Objects.equals("pet", i)).forEach(System.out::println);
	}

}
```
如果不要預設名稱為方法名稱，則可以如下設置 `@Bean(別名)`。
透過以下方式可以知道說 Bean 預設是單例的
```java=
        Pet one = run.getBean("pet", Pet.class);
		Pet two = run.getBean("pet", Pet.class);

		System.out.println(one == two);
```
*MyConfig* 本身也是一個組件，`@Configuration`  預設 `proxyBeanMethods` 是 true，代理 bean 的方法。

下面的方式可以證明說，外部無論對配置類中的某個組件註冊方法都是先前註冊容器的單例方法。其原因是 `proxyBeanMethods=true`，Spring boot 會檢查該組件是否在容器中。預設是保持單例，如果設定為 `false`，*每次調用都會產生一個新物件*，這表示會少了檢查是否存在容器中的步驟，可提升性能。
```java=
        MyConfig bean = run.getBean(MyConfig.class);
		User u1 = bean.user01();
		User u2 = bean.user01();
		System.out.println(u1 == u2);
```
2. @Import
給容器中自動創建出指定類型的組件，預設組件名稱是類名。
3. @Conditional
條件裝配，滿足 Conditional 指定的條件，則進行組件注入。其用法有如下圖中的等方法

![](https://i.imgur.com/zN5fLlj.png)

在底層經常使用。

4. @ImportResource
資源引入，像是導入 Spring 的配置檔案
```java=
@ImportResource(classpath:beans.xml)
```

## 配置綁定
1. @ConfigurationProperties
- prefix 是一個前綴參數
```java=
@Data
@ToString
@ConfigurationProperties(prefix = "tw")
@Component // 將此類添加到容器中
public class Car {
    private String brand;
    private Integer price;
}
```
接著我們在  application.properties 中就可以使用定義的配置
```
tw.brand=VOLVO
tw.price=100000
```
>只有在容器中的組件，才會有 Spring boot 提供的功能

透過簡易的 controller 設置，我們可以驗證是否存在我們設定的值
```java=
@RestController
public class Hello {
    @Autowired
    Car car;

    @GetMapping(value="/car")
    public Car getMethodName() {
        return car;
    }
    
}
// {"brand":"VOLVO","price":100000}
```
2. @EnableConfigurationProperties(CLASSNAME.class)
- 開啟指定類的配置綁定
- 把指定類的組件自動註冊到容器中
假設使用第三方套件時，其配置類沒有 `@Component` 將其注入到容器中時，我們可藉由此屬性配置

### 引導加載自動配置類
```java=
@SpringBootApplication 
相當於
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan("PACKAGE_NAME")
```
1. @SpringBootConfiguration 表示一個配置類
2. @ComponentScan 指定掃描那些
3. @EnableAutoConfiguration
EnableAutoConfiguration 是以下註解的合成
```java=
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class})
```
往下追會發現說其利用 `Register` 給容器中導入一系列組件，會從指定的 package 下，也就是 `@SpringBootApplication` 註解下該類 package。