---
title: Spring boot Web 開發
date: 2021-10-17
description: "Spring boot2"
tags: ["Spring boot"]
draft: true
---

Spring boot provides auto-configuration for Spring MVC that works well with most applications.

### 請求參數處理
#### 請求映射
- RequestMapping
    - value 請求路徑
    - method 使用的 HTTP Method
```java
    @RequestMapping(value="path", method=RequestMethod.GET)
    public SomeData requestMethodName(@RequestParam String param) {
        return new SomeData();
    }
```
在 Spring boot 中以下的請求映射都繼層於 `RequestMapping`
- GetMapping
- PostMapping
- DeleteMapping
- PutMapping

`DispatcherServlet` 是處理所有請求的開始，往上追最後會繼層一個 `HttpServlet`，當中請求會調用 `doGet` 方法（如果是 Get 請求），在 `DispatcherServlet` 中所有請求都會調用 `doDispatch(HttpServletRequest request, HttpServletResponse response)`。所有請求都存在於 `HandlerMapping` 中在 Spring boot 中有 5 種，Spring boot 自動配置歡迎頁面的 `WelcomePageHandlerMapping`，只要請求訪問 `/` 能訪問到 `index.html`。請求進來，會嘗試每個 `HandlerMapping` 查看是否有請求訊息，有就表示找到該請求對應的 `HandlerMapping`。

Spring boot 自動配置默認的 `RequestMappingHandlerMapping`其保存所有`@RequestMapping` 和 `handler` 映射規則，還有像是 `BeanNameUrlHandlerMapping`
- `RouterFunctionMapping` 和 `SimpleUrlHandlerMapping`

#### 普通參數與基本註解
- @PathVariable
```java
@GetMapping("/user/{id}")
public something getUser(@PathVariable("id") String id) {
    return something;
}
@GetMapping("/user/{id}/profile/{username}")
public something getUser(@PathVariable Map<String, String> pv) {
    // @PathVariable Map<String, String> pv 會將 id 和 username 封裝至 Map
    return something;
}
```
- @RequestHeader
    - 表示獲取 HTTP 請求中的某一個 Header
```java
@GetMapping("/user/{id}")
public something getUser(@PathVariable("id") String id, @RequestHeader("Authorization") String authorization) {
    return something;
}
```
```java
@GetMapping("/user/{id}")
public something getUser(@PathVariable("id") String id, @RequestHeader Map<String, String> headers) {
    // @RequestHeader Map<String, String> 獲取所有請求 Header
    return something;
}
```
- @RequestParam
    - 獲取 HTTP 請求 Path 中的參數，ex: `http://localhost:8080/user/{id}?age=18&inters=game&inters=baseball`
    - 用於 queryString 
```java
@GetMapping("/user/{id}?age=18&inters=game&inters=baseball")
public something getUser(@PathVariable("id") String id, @RequestParam("age") Integer age,@RequestParam("inters") List<String> inters) {
    // @RequestParam Map<String, String> 獲取所有請求參數
    return something;
}
```
- @CookieValue
    - 獲取 cookie 的值
```java
@GetMapping("/user/{id}")
public something getUser(@PathVariable("id") String id, @CookieValue("_ga") String _ga) {
    // @CookieValue("_ga") Cookie _ga 方式接收，表示獲取全部 cookie 內容
    return something;
}
```
- @RequestBody
    - 表示獲取 HTTP 請求中的 Body，通常建立資料或更新資料時 Body 才會有值
```java
@PostMapping("/user")
public something saveUser(@RequestBody User user) {
    return something;
}
``` 
- @RequestAttribute
    - 通常用於頁面轉發
```java
@GetMapping("/goto")
public String saveUser(HttpServletRequest request) {
    request.setAttribute("msg", "成功了...");
    request.setAttribute("code", "200");
    return "forward:/success";
}
@GetMapping("/success")
public Object success(@RequestAttribute("msg") String msg, @RequestAttribute("code") Integer code, HttpServletRequest request) {
    return request.getAttribute("msg");
}
``` 
- @MatrixVariable

##### 參數處理原理
- `HandlerMapping` 中找到能處理請求的 Handler (Controller.method)
- 為當前 `Handler` 找一個適配器 `HandlerAdapter`(RequestMappingHandlerAdapter)

`HandlerAdapter` 有以下四種方法
- RequestMappingHandlerAdapter
    - 支援方法上標示 `RequestMapping`
- HandlerFunctionAdapter
    - 支援函數式編程
- HttpRequestHandlerAdapter
- SimpleControllerHandlerAdapter

`HandlerMethodArgumentResolver` 是一個參數解析器，確定將要執行的目標方法每一個參數值是什麼，就是解析
- RequestParamMethodArgumentResolver
- PathVariableMethodArgumentResolver
- RequestResponseBodyMethodProcessor
- 等

`HandlerMethodReturnValueHandlerComposite` 處理返回值
- ModelAndViewMethodReturnValueHandler
- ModelMethodProcessor
- ResponseBodyEmitterReturnValueHandler
- 等

在我們的 controller 中的方法也是可以調用 Servlet API 
- WebRequest
- ServletRequest
- MultipartRequest
- HttpSession
- javax.servlet.http.PushBuilder
- Principal
- InputStream
- Reader
- HttpMethod
- Locale
- TimeZone
- ZoneId
- etc.
 
下面這個範例來說它會匹配 `ServletRequestMethodArgumentResolver` 這個解析器
```java
@GetMapping("/goto")
public String saveUser(HttpServletRequest request) {
    request.setAttribute("msg", "成功了...");
    request.setAttribute("code", "200");
    return "forward:/success";
}
```

在底層中，會符合 `ServletRequest.class` 這個物件
```java
    @Override
	public boolean supportsParameter(MethodParameter parameter) {
		Class<?> paramType = parameter.getParameterType();
		return (WebRequest.class.isAssignableFrom(paramType) ||
				ServletRequest.class.isAssignableFrom(paramType) ||
				MultipartRequest.class.isAssignableFrom(paramType) ||
				HttpSession.class.isAssignableFrom(paramType) ||
				(pushBuilder != null && pushBuilder.isAssignableFrom(paramType)) ||
				Principal.class.isAssignableFrom(paramType) ||
				InputStream.class.isAssignableFrom(paramType) ||
				Reader.class.isAssignableFrom(paramType) ||
				HttpMethod.class == paramType ||
				Locale.class == paramType ||
				TimeZone.class == paramType ||
				ZoneId.class == paramType);
	}
```

相較於複雜參數有可能會帶入這些
- Map
- Model（map、model 裡面的數據會被放在 request 的請求域 `request.setAttribute`）
- Errors/BindingResult
- RedirectAttributes（重定向攜帶數據）
- ServletResponse（response）
- SessionStatus
- UriComponentsBuilder
- ServletUriComponentsBuilder

下面的範例，是請求 `params` 接著轉發至 `success` 路徑，`Map<String, Object> map`、`Model model`、`HttpServletRequest request` 都可以在 request 請求中放數據
```java
    @GetMapping("/params")
    public String testParam(Map<String, Object> map, Model model, HttpServletRequest request, HttpServletResponse response) {
        map.put("Hello", "World");
        model.addAttribute("attributeName", "attributeValue");
        request.setAttribute("message", "Hello World");
        Cookie c = new Cookie("name", "value");
        c.setDomain("localhost");
        response.addCookie(c);
        return "forward:/success";
    }

    @GetMapping(value="/success")
    public Map success(HttpServletRequest request) {
        Object hello = request.getAttribute("Hello");
        Object attributeName  = request.getAttribute("attributeName");
        Object message = request.getAttribute("message");
        Map<String, Object> map = new HashMap<>();
        map.put("Hello", hello);
        map.put("attributeName", attributeName);
        map.put("message", message);
        return map;
    }
```

假設傳入的參數是屬於自定義的話會被 `ServletModelAttributeMethodProcessor` 進行處理，並判斷類型是否為 `SimpleValue`，之後還會進入 `GenericConversionService`，將 request 帶來的參數的字串轉成指定類型。而我們也可以自定義一個 `Converter` 進行數據轉換。
```java
public static boolean isSimpleValueType(Class<?> type) {
    return (Void.class != type && void.class != type &&
        (ClassUtils.isPrimitiveOrWrapper(type) ||
        Enum.class.isAssignableFrom(type) ||
        CharSequence.class.isAssignableFrom(type) ||
        Number.class.isAssignableFrom(type) ||
        Date.class.isAssignableFrom(type) ||
        Temporal.class.isAssignableFrom(type) ||
        URI.class == type ||
        URL.class == type ||
        Locale.class == type ||
        Class.class == type));
}
```

## 響應 JSON
### jackson.jar 與 @ResponseBody
我們只要引入 `spring-boot-starter-web` 就會幫我們引入 `json` 相關套件。所以我們只要給予 `@ResponseBody` 就可以給前端返回 `json` 格式數據。如下範例
```java
@ResponseBody
@GetMapping(value="/user")
public User saveUser(User user) {
    user.setName("Itachi");
    user.setAge(18);
    return user;
}
```

在一個 API 上有請求處理也同樣有返回值處理(`HandlerMethodReturnValueHandlerComposite`)，同樣會用下面的方法找到合適的處理器進行處理。

```java
  @Override
  public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
      ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

    HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
    if (handler == null) {
      throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
    }
    handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
  }
```

而 `selectHandler` 合適的處理器有以下
- ModelAndViewMethodReturnValueHandler
- ModelMethodProcessor
- ViewMethodReturnValueHandler
- ResponseBodyEmitterReturnValueHandler
- 等

以上面 API 範例來說最後是使用 `RequestResponseBodyMethodProcessor` 處理器。接下來會透過 `handleReturnValue` 進行回應數據的處理，如下

```java
  @Override
  public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
      ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
      throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

    mavContainer.setRequestHandled(true);
    ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
    ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);
    writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
  }
```

`writeWithMessageConverters` 則會將數據處理為 json 格式。在一個 HTTP 請求 Header 中有一個欄位是 `Accept`，表示能接受的內容類型，在轉換 json 格式時會做一個協商，瀏覽器會以該 Header 說它能接收什麼類型的數據內容，接著服務器會依據自己的能力，決定服務器生產出什麼樣的內容類型數據，在 Spring MVC 會遍歷底層的 `HttpMessageConverter` 看誰能處理，以上面 API 範例來看，就是 User 對象轉 JSON 格式；或是 JSON 轉 User 物件。默認 `HttpMessageConverter` 有以下
- MappingJackson2HttpMessageConverter
- ByteArrayHttpMessageConverter
- ResourceHttpMessageConverter
    - 真對檔案下載等
- StringHttpMessageConverter
- 等

其中 `MappingJackson2HttpMessageConverter` 救世會將物件轉 JSON 格式的處理。整體來說如下
```
@ResponseBody -----> RequestResponseBodyMethodProcessor -----> HttpMessageConverter(協商) -----> 轉成對應的資源
```

而協商會根據客戶端接收能力不同，而返回不同類型數據。也就是請求 Header 的 accept 值，如果 `*/*` 表示所有都可接受預設是 JSON，如要 XML 就 `application/xml`，這過程會和服務端的 10 種能力(JSON、XML) 等進行最佳匹配。

在 Spring MVC 可以使用 `spring.contentnegotiation.favor-parameter=true` 使用 `format` 進行回應資料的轉換，`http://localhost/user?format=xml` 或是 `http://localhost/user?format=json`。在底層協商部分會增加基於 `ParameteContentNegotiationStrategy` 的策略，因此協商內容不只是依據 HTTP 的 Header 內容。

### 自定義 HttpMessageConverter
從上面解析來看
1. `@ResponseBody` 響應出去調用 `RequestResponseBodyMethodProcessor` 處理
2. `Processor` 處理方法返回值，透過 `HttpMessageConverter` 處理
3. 所有 `HttpMessageConverter` 可支持各種 `Media-Type` 類型讀寫操作，最後找到合適的 `HttpMessageConverter`

如果要自定義可使用 `WebMvcConfigurer` 物件並覆寫 `extendMessageConverters(List<HttpMessageConverter<?>> converters)`。要以 `format` 方式進行處理，則可以透過覆寫`configureContentNegotiation(ContentNegotiationConfigurer configurer)`，在這之中如果只針對 `ParameterContentNegotiationStrategy` 進行配置，則 HTTP 的 Header 的可接受數據內容則會失效，因此最好也將 `HeaderContentNegotiationStrategy` 進行覆寫。