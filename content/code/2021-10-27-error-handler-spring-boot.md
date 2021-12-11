---
title: Spring boot Web 開發 - Error Handler
date: 2021-10-27
description: "Spring boot2"
tags: ["Spring boot"]
draft: false
---

從官方介紹內容來看錯誤處理內從大概是以下
1. 默認規則
- 預設下，Spring boot 提供 `/error` 處理所有錯誤的映射
- 對於非瀏覽器類型，會以 `JSON` 進行回應，其中包含錯誤、HTTP 狀態和異常訊息；對於瀏覽器則會以 *whilelabel* 進行視圖回應，會以 HTML 方式呈現
- 要自定義，添加 **View 解析為 Error**
- 要完全替代預設的行為，可以實作 `ErrorController` 並註冊該類型的 Bean 定義，或添加 `ErrorAttributes` 類型的組件以替換內容

異常整體流程處理
1. 執行目標方法，只要期間有錯誤都會被 JAVA 的 `catch` 語法給抓到，並將當前請求結束，其過程會進入 `dispatchException`
2. 進入 `processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException)`
3. 當中 mv = processHandlerException，處理 handler 發生的異常，接著回傳 *ModelAndView*，過程會遍歷 `handlerExceptionResolvers`，看誰能處理，系統默認的解析起有以下
- DefaultErrorAttributes
    - 先來處理異常。把異常訊息給 `request`，並返回 `null`
- HandlerExceptionResolveComposite
    - ExceptionHandlerExceptionResolver
    - ResponseStatusExceptionResolver
    - DefaultHandlerExceptionResolver
預設下沒有東西可以處理異常，因此異常被拋出，最後發送 `/error` 請求，被底層 `BasicErrorController` 處理。

## 定制錯誤邏輯
- 自定義錯誤頁面
    - 在 error 目錄下進行匹配
- @ControllerAdvice+@ExceptionHandler
    - 底層是 `ExceptionHandlerExceptionResolver`
- @ResponseStatus+自定義異常
    - 底層是 `ResponseStatusExceptionResolver`，把 `@ResponseStatus` 註解訊息底層調用 `response.sendError(statusCode, resolvedReason)`，tomcat 發送的 `/error`
- Spring 底層異常，如參數類型轉換
    - `DefaultHandlerExceptionResolver` 處理框架的異常
    - `response.sendError (HttpServletResponse.SC_BAD_REQUEST, ex.getMessage())`
- 自定義實現 `HandlerExceptionResolver` 異常處理

### @ControllerAdvice+@ExceptionHandler
```java
@RestControllerAdvice
@Slf4j
public class ExceptionHandlerAdvice {

    @ExceptionHandler(Exception.class) // 捕獲的異常，執行這個類別下對應的動作
    public ResponseResult<Void> handleException(Exception e) {
        log.error(e.getMessage(), e);
        return new ResponseResult<Void>(ResponseCode.SERVICE_ERROR.getCode(), ResponseCode.SERVICE_ERROR.getMsg(), null);
    }

    @ExceptionHandler(RuntimeException.class)
    public ResponseResult<Void> handleRuntimeException(RuntimeException e) {
        log.error(e.getMessage(), e);
        return new ResponseResult<Void>(ResponseCode.SERVICE_ERROR.getCode(), ResponseCode.SERVICE_ERROR.getMsg(), null);
    }

    @ExceptionHandler(BaseException.class)
    public ResponseResult<Void> handleBaseException(BaseException e) {
        log.error(e.getMessage(), e);
        ResponseCode code = e.getCode();
        return new ResponseResult<Void>(code.getCode(), code.getMsg(), null);
    }

}
```

### @ResponseStatus+自定義異常
```java
@ResponseStatus(value = HttpStatus.FORBIDDEN, reason = "User too many")
public UserTooManyException extends RuntimeException {
    public UserTooManyException() {

    }
    public UserTooManyException(String message) {
        super(message);
    }
}

@GetMapping("/user/all")
public something getUser() {
    throw new UserTooManyException();
    return something;
}
```
[可參考文章](https://www.baeldung.com/spring-response-status)

### HandlerExceptionResolver 自定義

```java
@Order(value = Ordered.HIGHEST_PRECEDENCE)
@Component // 放在容器中
public class CustomHandlerExceptionResolver implements HandlerExceptionResolver{
    @Override
    public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler,
            Exception ex) {
        // TODO Auto-generated method stub
        try {
            response.sendError(510, "Error...");
        } catch (Exception e) {
            //TODO: handle exception
        }
        return new ModelAndView();
    }
}

```
透過 Oeder 的註解讓當前異常處理優先權為最高。