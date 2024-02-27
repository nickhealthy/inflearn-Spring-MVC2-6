# 인프런 강의

해당 저장소의 `README.md`는 인프런 김영한님의 SpringBoot 강의 시리즈를 듣고 Spring 프레임워크의 방대한 기술들을 복기하고자 공부한 내용을 가볍게 정리한 것입니다.

문제가 될 시 삭제하겠습니다.



## 해당 프로젝트에서 배우는 내용

* 섹션 8 | 예외 처리와 오류 페이지
* 섹션 9 | API 예외 처리



# 섹션 8 | 예외 처리와 오류 페이지

## 서블릿 예외 처리 - 시작

예외처리를 하려면 서블릿 컨테이너가 예외를 어떻게 처리해야하는지 알아야한다.



#### 서블릿은 다음 2가지 방식으로 예외 처리를 지원한다.

* Exception(예외)
* response.sendError(HTTP 상태 코드, 오류 메시지)



### Exception(예외)

#### 자바 직접 실행

자바의 메인 메서드가 실행 도중 예외를 잡지 못하고 처음 실행한 `main()` 메서드를 넘어서 예외가 던져지면, 예외 정보를 남기고 해당 쓰레드는 종료된다.



#### 웹 애플리케이션

웹 애플리케이션은 사용자 요청 별로 별도의 쓰레드가 할당되고, 서블릿 컨테이너 안에서 실행된다.
만약 애플리케이션에서 예외를 잡지 못하고, 서블릿 밖으로까지 예외가 전달되면 아래와 같이 프로세스가 이루어진다.

* WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)

<u>결국 톰캣 같은 WAS까지 예외가 전달된다.</u>



#### 예제

먼저 WAS가 처리하는 방식을 알아보기 위해 스프링 부트가 제공하는 기본 예외 페이지를 꺼두자

[`application.properties`]

```java
server.error.whitelabel.enabled=false
```



[서블릿 예외 컨트롤러]

* `Exception`의 경우 서버 내부에서 처리할 수 없는 오류가 발생한 것으로 생각해서 HTTP 상태 코드를 500을 반환한다.

```java
package hello.exception.servlet;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;

@Slf4j
@Controller
public class ServletExController {

    @GetMapping("error-ex")
    public void errorEx() {
        throw new RuntimeException("예외 발생");
    }
}

```



### response.sendError(HTTP 상태 코드, 오류 메시지)

오류가 발생했을 때 `HttpServletResponse`가 제공하는 `sendError`라는 메서드를 사용해도 된다.
이것을 사용하면 당장 예외가 발생하는 것은 아니지만, <u>서블릿 컨테이너에게 오류가 발생했다는 것을 전달할 수 있다.</u>

* response.sendError(HTTP 상태 코드)
* response.sendError(HTTP 상태 코드, 오류 메시지)



#### 예제

[response.sendError]

* 해당 메서드를 호출하면 `response` 내부에 오류가 발생했다는 상태를 저장함
* 그리고 서블릿 컨테이너는 클라이언트에게 응답 전 response에 `sendError()`가 호출되었는지 확인 후, <u>호출 되었다면 설정한 오류 코드에 맞추어 기본 오류 페이지를 보여준다.</u>



#### sendError 흐름

* WAS(sendError 호출 기록 확인) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(`response.sendError()`)



## 서블릿 예외 처리 - 오류 화면 제공

과거에는 `web.xml` 파일에 오류 화면을 등록했지만, <u>지금은 스프링 부트를 통해 서블릿 컨테이너를 실행하기 때문에, 스프링 부트가 제공하는 기능을 사용해서 서블릿 오류 페이지를 등록하면 된다.</u>



### 예제



#### 처리 프로세스

[서블릿 오류 페이지 등록]

1. 예외나, `response.sendError`가 발생하게 되면 해당 컨트롤러를 타게 된다.
2. 이후 `ErrorPage` 클래스를 통해 **등록된 에러 컨트롤러 경로로 서버 내부에서 재요청을 하게 된다.**
   * WAS는 해당 예외를 처리하는 오류 페이지 정보를 확인한다.

```java
package hello.exception;

import org.springframework.boot.web.server.ConfigurableWebServerFactory;
import org.springframework.boot.web.server.ErrorPage;
import org.springframework.boot.web.server.WebServerFactory;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;


/**
 * 서블릿 오류 페이지 등록
 */
@Component
public class WebServerCustomizer implements WebServerFactoryCustomizer<ConfigurableWebServerFactory> {

    @Override
    public void customize(ConfigurableWebServerFactory factory) {
        // `response.sendError(404)` : `errorPage404` 호출
        ErrorPage errorPage404 = new ErrorPage(HttpStatus.NOT_FOUND, "/error-page/404");

        // `response.sendError(500)` : `errorPage500` 호출
        ErrorPage errorPage500 = new ErrorPage(HttpStatus.INTERNAL_SERVER_ERROR, "/error-page/500");

        // `RuntimeException` 또는 그 자식 타입의 예외: `errorPageEx` 호출
        ErrorPage errorPageEx = new ErrorPage(RuntimeException.class, "/error-page/500");
        factory.addErrorPages(errorPage404, errorPage500, errorPageEx);
    }
}
```



[오류를 처리할 컨트롤러]

```java
package hello.exception.servlet;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

@Slf4j
@Controller
public class ErrorPageController {

    @RequestMapping("/error-page/404")
    public String errorPage404(HttpServletRequest request, HttpServletResponse response) {
        log.info("errorPage 404");
        return "error-page/404";
    }

    @RequestMapping("/error-page/500")
    public String errorPage500(HttpServletRequest request, HttpServletResponse response) {
        log.info("errorPage 500");
        return "error-page/500";
    }
}
```



## 서블릿 예외 처리 - 오류 페이지 작동 원리

### 예외 발생과 오류 페이지 요청 흐름

1. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
2. WAS `/error-page/500` 다시 요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/error-page/500) -> View



**중요한 점은 클라이언트는 서버 내부에서 이런 일이 일어나는지 전혀 알지 못한다. 오직 서버 내부에서 오류 페이지를 찾기 위해 추가적인 호출을 한다.**



#### 중간 정리

1. 예외가 발생해서 WAS까지 전파된다.
2. WAS는 오류 페이지 경로를 찾아 내부에서 오류 페이지를 호출한다. 이때 오류 페이지 경로로 필터, 서블릿, 인터셉터, 컨트롤러가 모두 다시 호출된다.



#### 오류 정보 추가

**WAS는 오류 페이지를 단순히 다시 요청만 하는 것이 아니라, 오류 정보를 `request.attribute`에 추가해서 넘겨준다.**

> 이외에도 다양한 정보를 넘겨준다.

[오류를 처리할 컨트롤러]

```java
package hello.exception.servlet;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

@Slf4j
@Controller
public class ErrorPageController {

    public static final String ERROR_EXCEPTION = "jakarta.servlet.error.exception";
    public static final String ERROR_EXCEPTION_TYPE = "jakarta.servlet.error.exception_type";
    public static final String ERROR_MESSAGE = "jakarta.servlet.error.message";
    public static final String ERROR_REQUEST_URI = "jakarta.servlet.error.request_uri";
    public static final String ERROR_SERVLET_NAME = "jakarta.servlet.error.servlet_name";
    public static final String ERROR_STATUS_CODE = "jakarta.servlet.error.status_code";

    @RequestMapping("/error-page/404")
    public String errorPage404(HttpServletRequest request, HttpServletResponse response) {
        log.info("errorPage 404");
        printErrorInfo(request);
        return "error-page/404";
    }

    @RequestMapping("/error-page/500")
    public String errorPage500(HttpServletRequest request, HttpServletResponse response) {
        log.info("errorPage 500");
        printErrorInfo(request);
        return "error-page/500";
    }

    private void printErrorInfo(HttpServletRequest request) {
        log.info("ERROR_EXCEPTION : {}", request.getAttribute(ERROR_EXCEPTION));
        log.info("ERROR_EXCEPTION_TYPE : {}", request.getAttribute(ERROR_EXCEPTION_TYPE));
        log.info("ERROR_MESSAGE : {}", request.getAttribute(ERROR_MESSAGE));
        log.info("ERROR_REQUEST_URI : {}", request.getAttribute(ERROR_REQUEST_URI));
        log.info("ERROR_SERVLET_NAME : {}", request.getAttribute(ERROR_SERVLET_NAME));
        log.info("ERROR_STATUS_CODE : {}", request.getAttribute(ERROR_STATUS_CODE));
        log.info("dispatchType : {}", request.getDispatcherType());
    }
}

```



## 서블릿 예외 처리 - 필터

#### 목표

<u>서블릿이 제공</u>하는 `DispatchType` 이해하기



#### 예외 발생과 오류 페이지 요청 흐름

1. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
2. WAS `/error-page/500` 다시 요청 -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러(/error-page/500) -> View



#### 예외 발생 및 오류 페이지 처리를 위한 재호출의 문제점

오류가 발생하면 위와 같이 WAS 내부에서 필터, 인터셉터 등이 다시 호출된다.
하지만 이미 로그인 인증 체크 같은 경우 이미 처음 클라이언트가 요청했을 때 검증을 완료했는데 다시 한번 호출되는 것은 비효율적이다. 
**따라서 클라이언트로부터 발생한 정상 요청인지, 아니면 오류 페이지를 출력하기 위한 내부 요청인지 구분할 수 있어야 한다.**
서블릿은 이런 문제를 해결하기 위해 `DisPatcherType`이라는 추가 정보를 제공한다.



### DispatcherType

필터는 이런 경우를 위해 `DispatcherType` 옵션을 제공한다.

```java
public enum DispatcherType {
     FORWARD, // 서블릿에서 다른 서블릿이나 JSP를 호출할 때
     INCLUDE, // 서블릿에서 다른 서블릿이나 JSP의 결과를 포함할 때
     REQUEST, // 클라이언트 요청
     ASYNC, // 서블릿 비동기 호출
     ERROR // 오류 요청
}
```



[필터 등록 - WebConfig]

* `filterRegistrationBean.setDispatcherTypes(DispatcherType.REQUEST,  DispatcherType.ERROR);` 
  * <u>해당 메서드를 통해 어떤 타입의 요청인지를 구분해서 처리할 수 있다.</u> 
  * 기본 값은 `REQUEST`이므로, <u>별도로 해당 메서드를 처리해주지 않는 이상 필터는 클라이언트의 요청만 처리한다.</u>

```java
package hello.exception;

import hello.exception.filter.LogFilter;
import jakarta.servlet.DispatcherType;
import jakarta.servlet.Filter;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class WebConfig {

    @Bean
    public FilterRegistrationBean logFilter() {
        FilterRegistrationBean<Filter> filterRegistrationBean = new FilterRegistrationBean<>();
        filterRegistrationBean.setFilter(new LogFilter());
        filterRegistrationBean.setOrder(1);
        filterRegistrationBean.addUrlPatterns("/*");
        filterRegistrationBean.setDispatcherTypes(DispatcherType.REQUEST, DispatcherType.ERROR);

        return filterRegistrationBean;
    }
}
```



## 서블릿 예외 처리 - 인터셉터



### 예제 - 인터셉터 중복 호출 제거

[인터셉터 구현]

* 로그에 `request.getDispatcherType` 추가

```java
package hello.exception.interceptor;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import java.util.UUID;

@Slf4j
public class LogInterceptor implements HandlerInterceptor {

    public static final String LOG_ID = "logId";

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String requestURI = request.getRequestURI();
        String uuid = UUID.randomUUID().toString();
        request.setAttribute(LOG_ID, uuid);
        // request.getDispatcherType 추가
        log.info("REQUEST  [{}][{}][{}][{}]", uuid, request.getDispatcherType(), requestURI, handler);
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        log.info("postHandle [{}]", modelAndView);
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        String requestURI = request.getRequestURI();
        String logId = (String) request.getAttribute(LOG_ID);
        // request.getDispatcherType 추가
        log.info("RESPONSE [{}][{}][{}]", logId, request.getDispatcherType(), requestURI);
        if (ex != null) {
            log.error("afterCompletion error!!", ex);
        }
    }

}
```



[인터셉터 등록 - WebConfig]

* 앞서 필터에서는 서블릿이 제공하는 `DispatcherType` 이용해 어떤 타입인 경우에 따라 필터를 적용할 지 선택할 수 있었지만, 인터셉터는 스프링이 제공하는 기능이므로 `DispatcherType`과 무관하게 항상 호출된다.
* 대신, 오류 페이지 경로를 `excludePathPatterns`를 사용해서 빼주면 된다.

```java
package hello.exception;

import hello.exception.filter.LogFilter;
import hello.exception.interceptor.LogInterceptor;
import jakarta.servlet.DispatcherType;
import jakarta.servlet.Filter;
import org.springframework.boot.web.servlet.FilterRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LogInterceptor())
                .order(1)
                .addPathPatterns("/**") // 전체 경로
                .excludePathPatterns(
                        "/css/**", "/*.ico",
                        "/error", "/error-page/**"
                ); // 오류 페이지 경로
    }

//    @Bean
    public FilterRegistrationBean logFilter() {
        FilterRegistrationBean<Filter> filterRegistrationBean = new FilterRegistrationBean<>();
        filterRegistrationBean.setFilter(new LogFilter());
        filterRegistrationBean.setOrder(1);
        filterRegistrationBean.addUrlPatterns("/*");
        filterRegistrationBean.setDispatcherTypes(DispatcherType.REQUEST, DispatcherType.ERROR);

        return filterRegistrationBean;
    }
}

```



### 전체 흐름 정리

1. `/hello` 정상 요청

   * WAS(/hello, dispatchType=REQUEST) -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러 -> View

2. `/error-ex` 오류 요청

   * 필터는 `DispatchType` 으로 중복 호출 제거 ( `dispatchType=REQUEST` )

   * 인터셉터는 경로 정보로 중복 호출 제거( `excludePathPatterns("/error-page/**")` )

   * ```
     1. WAS(/error-ex, dispatchType=REQUEST) -> 필터 -> 서블릿 -> 인터셉터 -> 컨트롤러
     2. WAS(여기까지 전파) <- 필터 <- 서블릿 <- 인터셉터 <- 컨트롤러(예외발생)
     3. WAS 오류 페이지 확인
     4. WAS(/error-page/500, dispatchType=ERROR) -> 필터(x) -> 서블릿 -> 인터셉터(x) -> 컨트
     롤러(/error-page/500) -> View
     ```



## 스프링 부트 - 오류 페이지1

지금까지의 과정을 보면 다음과 같다.

* `WebServerCustomizer`를 만들어 에러 목록들을 만들고,
* 예외 종류에 따라 `ErrorPage`를 추가하고,
* 예외 처리용 컨트롤러 `ErrorPageController`를 만들었다.



#### 스프링부트는 이런 과정을 모두 기본으로 제공한다.

* `ErrorPage`를 자동으로 등록한다. 이때 `/error`라는 경로로 기본 오류 페이지를 설정한다.
  * `new ErrorPage("/error")`, 상태코드와 예외를 설정하지 않으면 기본 오류 페이지로 사용된다.
  * 서블릿 밖으로 예외가 발생하거나, `response.sendError(...)`가 호출되면 모든 오류는 `/error`를 호출한다.
* [오류를 처리할 컨트롤러] - `BasicErrorController`라는 스프링 컨트롤러를 자동으로 등록한다.
  * `ErrorPage`에서 등록한 `/error`를 매핑해서 처리하는 컨트롤러다.



#### 뷰 선택 우선순위

스프링부트가 에러페이지를 보여주는 롤은 따로 정해져 있는데 다음과 같다.

`BasicErrorController` 의 처리 순서

1. 뷰 템플릿
   * `resources/templates/error/500.html`
   * `resources/templates/error/5xx.html`
2. 정적 리소스(static, public)
   * `resources/static/error/400.html`
   * `resources/static/error/404.html`
   * `resources/static/error/4xx.html`
3. 적용 대상이 없을 때 뷰 이름(error)
   * `resources/templates/error.html`



### 정리

**개발자는 오류 페이지만 등록해하면 오류페이지를 구현할 수 있다.** 



## 스프링 부트 - 오류 페이지2

### BasicErrorController가 제공하는 기본 정보들

`BasicErrorController` 컨트롤러는 다음 정보를 model에 담아서 뷰에 전달한다. 
뷰 템플릿은 이 값을 활용해서 출력할 수 있다.

```
* timestamp: Fri Feb 05 00:00:00 KST 2021
* status: 400
* error: Bad Request
* exception: org.springframework.validation.BindException * trace: 예외 trace
 * message: Validation failed for object='data'. Error count: 1
 * errors: Errors(BindingResult)
* path: 클라이언트 요청 경로 (`/hello`)
```



클라이언트에게 해당 정보들을 노출하는 것은 좋치 않으므로 스프링에서는 `BasicErrorController`의 오류 컨트롤러에서 다음 오류 정보를 `model`에 포함할지 선택할 수 있다.

[`application.properties`]

```
# exception 포함 여부
server.error.include-exception=false
# message 포함 여부
server.error.include-message=never
# trace 포함 여부
server.error.include-stacktrace=never
# errors 포함 여부
server.error.include-binding-errors=never
```



#### 스프링 부트 오류 관련 옵션

* `server.error.whitelabel.enabled=true` : 오류 처리 화면을 못 찾을 시, 스프링 whitelabel 오

  류 페이지 적용

* `server.error.path=/error` : 오류 페이지 경로, 스프링이 자동 등록하는 서블릿 글로벌 오류 페이지 경로와 `BasicErrorController` 오류 컨트롤러 경로에 함께 사용된다.



#### 확장 포인트

* 에러 공통 처리 컨트롤러의 기능을 변경하고 싶으면 `ErrorController, BasicErrorController`의 상속 받아 기능을 구현하면 된다.



# 섹션 9 | API 예외 처리

## API 예외 처리 - 시작

API 방식으로 요청했을 때 에러가 발생하게 되면, HTML이 아닌 API의 JSON 형식으로 데이터가 반환되어야한다.



### 예제

[`ErrorPageConntroller`]

* `produces = MediaType.APPLICATION_JSON_VALUE` 의 뜻은 클라이언트가 요청하는 HTTP Header의 Accept 의 값이 `application/json` 일 때 해당 메서드가 호출된다는 것이다

```java
/**
 * 파라미터 produces: 클라이언트가 요청하는 HTTP Header의 Accept 값이 application/json일 때 해당 메서드가 호출된다.
 */
@RequestMapping(value = "/error-page/500", produces = MediaType.APPLICATION_JSON_VALUE)
public ResponseEntity<Map<String, Object>> errorPage500Api(HttpServletRequest request, HttpServletResponse response) {
    log.info("API errorPage 500");

    Map<String, Object> result = new HashMap<>();
    Exception ex = (Exception) request.getAttribute(ERROR_EXCEPTION);
    result.put("status", request.getAttribute(ERROR_STATUS_CODE));
    result.put("message", ex.getMessage());

    Integer statusCode  = (Integer) request.getAttribute(RequestDispatcher.ERROR_STATUS_CODE);
    return new ResponseEntity<>(result, HttpStatus.valueOf(statusCode));

}
```



## API 예외 처리 - 스프링 부트 기본 오류 처리

API 예외 처리도 스프링부트가 제공하는 기본 오류 방식을 사용할 수 있다
`BasicErrorController`가 동일하게 그 역할을 수행한다.

* errorHtml은 클라이언트의 Accept 헤더 값이 `text/html`인 경우 view를 제공한다.

```java
 @RequestMapping(produces = MediaType.TEXT_HTML_VALUE)
 public ModelAndView errorHtml(HttpServletRequest request, HttpServletResponse
 response) {}
 @RequestMapping
 public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {}
```



### 스프링부트가 제공하는 기본 오류 방식 - BasicErrorController

테스트를 위해 `WebServerCustomizer`의 `@Component` 주석 처리 후 테스트를 진행하자.

* <u>스프링 부트는 `BasicErrorController` 가 제공하는 기본 정보들을 활용해서 오류 API를 생성해준다.</u>
* `BasicErrorController`를 확장하면 JSON 메시지도 변경할 수 있지만 나중에 나올 `@ExceptionHandler`를 사용하는 것이 더 나은 방법이다.
* <u>따라서 이 방법은 HTML 화면을 처리할 때 사용하자</u>

```
{
    "timestamp": "2024-02-24T03:56:57.284+00:00",
    "status": 500,
    "error": "Internal Server Error",
    "path": "/api/members/ex"
}
```



## API 예외 처리 - HandlerExceptionResolver 시작

예외가 발생햇을 때 서블릿을 넘어 WAS까지 예외가 전달되면 HTTP 상태코드가 무조건 500으로 처리된다.
<u>발생하는 예외에 따라 400, 404 등 다른 상태로 처리하고, 오류 메시지나 형식을 다르게 처리하고 싶을 때 사용한다.</u>



### HandlerExceptionResolver

스프링 MVC는 컨트롤러(핸들러) 밖으로 예외가 던져진 경우 예외를 해결하고, 동작을 새로 정의할 수 있는 방법을 제공한다.
컨트롤러 밖으로 던져진 에러를 해결하고, 동작하는 방식을 변경하고 싶으면 `HandlerExceptionResolver`를 사용하면된다.



<img width="748" alt="스크린샷 2024-02-25 오후 1 22 05" src="https://github.com/nickhealthy/inflearn-Spring-MVC2-6/assets/66216102/8c0a3580-3a7c-41ff-82c4-f14784334a20">



#### HandlerExceptionResolver - 인터페이스

* `handler` : 핸들러(컨트롤러) 정보
*  `Exception ex` : 핸들러(컨트롤러)에서 발생한 발생한 예외

```java
public interface HandlerExceptionResolver {
  ModelAndView resolveException (HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex);
}
```



### 예제

만약 사용자가 입력을 잘못하여  `IllegalArgumentException` 예외가 발생하게 되면 500에러가 아닌, 400에러로 바꾸는 예제이다.

* <u>`ExceptionResolver` 가 `ModelAndView` 를 반환하는 이유는 마치 try, catch를 하듯이, `Exception` 을 처리해서 정상 흐름 처럼 변경하는 것이 목적이다.</u> 이름 그대로 `Exception` 을 Resolver(해결)하는 것이 목적이다.

```java
package hello.exception.resolver;

import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.servlet.HandlerExceptionResolver;
import org.springframework.web.servlet.ModelAndView;

@Slf4j
public class MyHandlerExceptionResolver implements HandlerExceptionResolver {

    @Override
    public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        try {
            if (ex instanceof IllegalAccessException) {
                log.info("IllegalAccessException resolver to 400");
                response.sendError(HttpServletResponse.SC_BAD_REQUEST, ex.getMessage());

                return new ModelAndView();
            }
        } catch (Exception e) {
            log.error("resolver ex", e);
        }

        return null;
    }
}
```



* `WebMvcConfigurer`를 통해 등록

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void extendHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) {
        resolvers.add(new MyHandlerExceptionResolver());
    }
}
```



#### 결과 값

```
{
    "timestamp": "2024-02-25T04:33:51.209+00:00",
    "status": 400,
    "error": "Bad Request",
    "path": "/api/members/bad"
}
```



#### 반환 값에 따른 동작 방식

`HandlerExceptionResolver` 의 반환 값에 따른 **`DispatcherServlet` 의 동작 방식은 다음과 같다.**

* 빈 ModelAndView: 뷰를 렌더링 하지 않고, 정상 흐름으로 서블릿이 리턴된다.
* ModelAndVIew: 뷰를 렌더링 한다.
* null: null을 반환하면, 다음 `ExceptionResolver`를 찾아 실행한다. 만약 처리할 수 있는 `ExceptionResolver` 가 없으면 예외 처리가 안되고, 기존에 발생한 예외를 서블릿 밖으로 던진다.



#### ExceptionResolver 활용

* 예외 상태 코드 변환
  * 예외를 `response.sendError(xxx)` 호출로 변경해서 서블릿에서 상태 코드에 따른 오류를 처리 하도록 위임
  * 이후 WAS는 서블릿 오류 페이지를 찾아서 내부 호출, 예를 들어서 스프링 부트가 기본으로 설정한 `/ error` 가 호출됨
    * <u>방금 위의 예제에서는 `BasicErrorController`를 사용해서 오류 페이지를 처리함</u>

* 뷰 템플릿 처리
  *  `ModelAndView` 에 값을 채워서 예외에 따른 새로운 오류 화면 뷰 렌더링 해서 고객에게 제공

* API 응답 처리
  *  `response.getWriter().println("hello");` 처럼 HTTP 응답 바디에 직접 데이터를 넣어주는 것도 가능하다. 여기에 JSON 으로 응답하면 API 응답 처리를 할 수 있다.



## API 예외 처리 - HandlerExceptionResolver 활용

#### 예외를 ExceptionResolver에서 마무리 하기

이때까지 예외가 발생하게 되면 WAS까지 예외가 던져지고, WAS에서 오류 페이지 정보를 찾아 다시 `/error`(스프링부트가 기본으로 사용하는 오류 페이지 경로)를 호출하는 과정은 복잡하다.
<u>`ExceptionResolver`를 활용하면 예외가 발생하였을 때 여기서 바로 해결할 수 있다.</u>



### 예제

* 사용자 오류 정의

```java
package hello.exception.exception;

public class UserException extends RuntimeException {
    public UserException() {
        super();
    }

    public UserException(String message) {
        super(message);
    }

    public UserException(String message, Throwable cause) {
        super(message, cause);
    }

    public UserException(Throwable cause) {
        super(cause);
    }

    protected UserException(String message, Throwable cause, boolean enableSuppression, boolean writableStackTrace) {
        super(message, cause, enableSuppression, writableStackTrace);
    }
}
```



* 예외 컨트롤러 추가

```java
package hello.exception.api;

import hello.exception.exception.UserException;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@Slf4j
@RestController
public class ApiExceptionController {

    @GetMapping("/api/members/{id}")
    public MemberDto getMember(@PathVariable("id") String id) {
				...
        if (id.equals("user-ex")) {
            throw new UserException("사용자 오류");
        }
				
        return new MemberDto(id, "hello" + id);
    }

    @Data
    @AllArgsConstructor
    static class MemberDto {
        private String memberId;
        private String name;
    }
}

```



* `UserException`를 처리하는 `ExceptionResolver`
  * `String acceptHeader = request.getHeader("accept");` 를 통해 헤더 정보를 보고 JSON, HTML를 적절하게 처리해준다.
  * 응답데이터를 HTTP 바디에 직접 넣어주기 위해 `response.메소드`를 사용해서 처음 서블릿의 응답을 사용할 때처럼 구현되었다.(인터페이스를 구현해서 사용하므로 `ModelAndView` return을 해결할 수 없다.)

```java
package hello.exception.resolver;

import com.fasterxml.jackson.databind.ObjectMapper;
import hello.exception.exception.UserException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.servlet.HandlerExceptionResolver;
import org.springframework.web.servlet.ModelAndView;

import java.util.HashMap;
import java.util.Map;

@Slf4j
public class UserHandlerExceptionResolver implements HandlerExceptionResolver {

    private final ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        try {
            if (ex instanceof UserException) {
                log.info("UserException resolver to 400");
                String acceptHeader = request.getHeader("accept");
                response.setStatus(HttpServletResponse.SC_BAD_REQUEST);

                if ("application/json".equals(acceptHeader)) {
                    Map<String, Object> errorResult = new HashMap<>();
                    errorResult.put("ex", ex.getClass());
                    errorResult.put("message", ex.getMessage());

                    String result = objectMapper.writeValueAsString(errorResult);

                    response.setContentType("application/json");
                    response.setCharacterEncoding("utf-8");
                    response.getWriter().write(result);
                    return new ModelAndView();
                } else {
                    // TEXT/HTML
                    return new ModelAndView("error/400");
                }
            }
        } catch (Exception e) {
            log.error("resolver ex", e);
        }

        return null;
    }
}
```



#### 결과

* accept: application/json인 경우

```json
{
    "ex": "hello.exception.exception.UserException",
    "message": "사용자 오류"
}
```



* accept: text/html인 경우

```html
 <!DOCTYPE HTML>
 <html>
 ...
 </html>
```



#### 정리

* `ExceptionResolver`를 사용하면 컨트롤러에서 예외가 발생해도 예외는 이곳에서 모두 처리되고, 서블릿 컨테이너까지 예외가 전달되지 않는다. 서블릿은 정상 응답을 해주고 예외 처리가 끝이난다.
* 즉, 서블릿 컨테이너까지 예외가 올라가서 복잡하고 지저분하게 추가 프로세스가 처리되지 않는 것이다.



## API 예외 처리 - 스프링이 제공하는 ExceptionResolver1

스프링이 기본으로 제공하는 `ExceptionResolver`는 다음과 같다.
`HandlerExceptionResolverComposite` 에 다음 순서로 등록한다.

1. `ExceptionHandlerExceptionResolver`(가장 중요)
2. `ResponseStatusExceptionResolver`
3. `DefaultHandlerExceptionResolver` 우선순위가가장낮다.



#### ExceptionHandlerExceptionResolver

`@ExceptionHHandler`을 처리한다. API 예외 처리는 대부분 이 기능으로 해결한다.



#### ResponseStatusExceptionResolver

HTTP 상태 코드를 지정해준다.



#### DefaultHandlerExceptionResolver

스프링 내부 기본 예외를 처리한다.



### ResponseStatusExceptionResolver

`ResponseStatusExceptionResolver` 는 예외에 따라서 HTTP 상태 코드를 지정해주는 역할을 한다.

다음 두 가지의 경우를 처리한다.

* `@ResponseStatus`가 달려있는 예외
* `ResponseStatusException` 예외



#### 예제 - @ResponseStatus

* `BadRequestException` 예외가 컨트롤러 밖으로 넘어감녀 ResponseStatusExceptionResolver 예외가 해당 애노테이션을 확인해서 오류 코드를 400으로 변경하고, 메시지도 담는다.
  * `ResponseStatusExceptionResolver`도 결국 `response.sendError(statusCode, resolvedReason)`를 호출해 이전에 했던 실습과 똑같다.
  * <u>`sendError`를 호출했기 때문에 WAS에서 다시 오류 페이지를 내부에 요청한다.</u>
* MessageSource 기능도 가능하다.

```java
 package hello.exception.exception;
 import org.springframework.http.HttpStatus;
 import org.springframework.web.bind.annotation.ResponseStatus;
 
// @ResponseStatus(code = HttpStatus.BAD_REQUEST, reason = "잘못된 요청 오류") 
@ResponseStatus(code = HttpStatus.BAD_REQUEST, reason = "error.bad") // MessageSource 기능
public class BadRequestException extends RuntimeException {
}
```



* 결과

```json
 {
    "status": 400,
    "error": "Bad Request",
    "exception": "hello.exception.exception.BadRequestException",
    "message": "잘못된 요청 오류",
    "path": "/api/response-status-ex1"
}
```



#### 예제 - ResponseStatusException

위에서 살펴본 `@ResponseStatus`는 라이브러리 등 개발자가 직접 변경할 수 없는 예외에는 적용할 수 없다.
추가로 애노테이션을 사용하기 때문에 조건에 따라 분기 처리도 불가능하다.
이때 사용하는 것이 `ResponseStatusException` 예외를 사용하면 된다.

```java
 @GetMapping("/api/response-status-ex2")
 public String responseStatusEx2() {
     throw new ResponseStatusException(HttpStatus.NOT_FOUND, "error.bad", new
 IllegalArgumentException());
 }
```



## API 예외 처리 - 스프링이 제공하는 ExceptionResolver2

<u>`DefaultHandlerExceptionResolver`는 스프링 내부에서 발생하는 스프링 예외를 해결한다.</u>
대표적으로 파라미터 바인딩 시점에 타입이 맞지 않으면 내부에서 `TypeMismatchException`이 발생하게 되는데, 이 경우 예외가 발생했기 때문에 서블릿 컨테이너까지 오류가 올라가고, 500 오류가 발생하게 되지만, 해당 예외처리로 인해 HTTP 상태 코드를 400오류로 변경하게 된다.

* <u>해당 오류 처리도 `sendError(400)`를 호출했기 때문에 WAS에서 다시 오류 페이지를 내부에 요청한다.</u>



### 예제 - DefaultHandlerExceptionResolver

```java
 @GetMapping("/api/default-handler-ex")
 public String defaultException(@RequestParam Integer data) {
     return "ok";
 }
```



#### 실행결과

```json

{
    "timestamp": "2024-02-26T08:40:35.920+00:00",
    "status": 400,
    "error": "Bad Request",
    "message": "Failed to convert value of type 'java.lang.String' to required type 'java.lang.Integer'; For input string: \"hello\"",
    "path": "/api/default-handler-ex"
}
```



### 정리 - ExceptionResolver

1. `ResponseStatusExceptionResolver`: HTTP 응답 코드 변경
2. `DefaultHandlerExceptionResolver`: 스프링 내부 예외 처리
3. `ExceptionHandlerExceptionResolver`: 다음 강의에서..
   * `HandlerExceptionResolver`를 직접 구현해서 사용하는 것은 복잡헀음
   * ModelAndView를 반환해야 하는 것도 API 형식에는 맞지 않았음
   * 따라서 응답 바디에 데이터를 넣어주기 위해 `response.여러 메소드`를 사용하여(기존 서블릿 처리 방식처럼) 응답해줬음



## API 예외 처리 - @ExceptionHandler

<u>지금까지 살펴본 `BasicErrorController` 를 사용하거나 `HandlerExceptionResolver` 를 직접 구현하는 방식으로 API 예외를 다루기는 쉽지 않다.</u> 

* API는 각 시스템 마다 응답의 모양이 다르고, 스펙도 모두 다르다.
* 예외에 따라 각각의 다른 데이터를 출력해야 할 수도 있다.
* 같은 예외라 해도 어떤 컨트롤러에서 발생했는가에 따라 다른 예외 응답을 내려주어야 할 수 있다.



#### API 예외처리의 어려운 점

* `HandlerExceptionResolver`는 `ModelAndView`을 반환해야한다. 이것은 API 응답에 필요하지 않다.
  * 따라서 `HttpServletResponse`에 직접 응답 데이터를 넣어주었고, 빈 `ModelAndView`를 반환해서 정상처리 동작이 된 것처럼 만들어야 하는 수고가 있다.
* 특정 컨트롤러에서만 발생하는 예외를 별도로 처리하기 어렵다.



#### @ExceptionHandler

스프링은 API 예외 처리 문제를 해결하기 위해 `@ExceptionHandler`라는 애노테이션을 사용하는 매우 편리한 예외 처리 기능을 제공한다. 이것이 바로 `ExceptionHandlerExceptionResolver` 이다.



### 예제 - @ExceptionHandler

[`ErrorResult`]

* 예외가 발생했을 때 API 응답으로 사용하는 객체를 정의

```java
package hello.exception.exhandler;
import lombok.AllArgsConstructor;
import lombok.Data;

@Data
@AllArgsConstructor
public class ErrorResult {
   private String code;
   private String message;
}
```



[`ApiExceptionV2Controller`]

`@ExceptionHandler` 예외 처리 방법

* 해당 애너테이션을 선언하고, 컨트롤러에서 처리하고 싶은 예외를 지정해주면 된다. 
  * 해당 컨트롤럴에서 예외가 발생하면 이 메서드가 호출된다.

```java
package hello.exception.exhandler;

import hello.exception.exception.UserException;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@Slf4j
@RestController
public class ApiExceptionV2Controller {

    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(IllegalArgumentException.class)
    public ErrorResult illegalExHandle(IllegalArgumentException e) {
        log.error("[exceptionHandle] ex", e);
        return new ErrorResult("BAD", e.getMessage());
    }

    @ExceptionHandler
    public ResponseEntity<ErrorResult> userExHandle(UserException e) {
        log.error("[exceptionHandle] ex", e);
        ErrorResult errorResult = new ErrorResult("USER-EX", e.getMessage());
        return new ResponseEntity<>(errorResult, HttpStatus.BAD_REQUEST);
    }

    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    @ExceptionHandler
    public ErrorResult exHandle(Exception e) {
        log.error("[exceptionHandle] ex", e);
        return new ErrorResult("EX", "내부 오류");
    }


    @GetMapping("/api2/members/{id}")
    public MemberDto getMember(@PathVariable("id") String id) {

        if (id.equals("ex")) {
            throw new RuntimeException("잘못된 사용자");
        }

        if (id.equals("bad")) {
            throw new IllegalArgumentException("잘못된 입력 값");
        }

        if (id.equals("user-ex")) {
            throw new UserException("사용자 오류");
        }

        return new MemberDto(id, "hello" + id);
    }

    @Data
    @AllArgsConstructor
    static class MemberDto {
        private String memberId;
        private String name;
    }

}
```



### 참고

#### 다양한 예외

다음과 같이 다양한 예외를 한번에 처리 가능

```java
@ExceptionHandler({AException.class, BException.class})
public String ex(Exception e) {
   log.info("exception e", e);
}
```



#### 예외 생략

* `@ExceptionHandler`에 예외를 생략 가능하다. 생략하면 메서드 파라미터의 예외가 지정된다.

```java
@ExceptionHandler
public ResponseEntity<ErrorResult> userExHandle(UserException e) {}
```



#### HTML 오류 화면

다음과 같이 `ModelAndView` 를 사용해서 오류 화면(HTML)을 응답하는데 사용할 수도 있다.

```java
@ExceptionHandler(ViewException.class)
public ModelAndView ex(ViewException e) {
   log.info("exception e", e);
   return new ModelAndView("error");
}
```



### 실행 흐름 예시

```java
@ResponseStatus(HttpStatus.BAD_REQUEST)
@ExceptionHandler(IllegalArgumentException.class)
public ErrorResult illegalExHandle(IllegalArgumentException e) {
   log.error("[exceptionHandle] ex", e);
   return new ErrorResult("BAD", e.getMessage());
}
```

* 컨트롤러를 호출한 결과 `IllegalArgumentException` 예외가 컨트롤러 밖으로 던져진다.
* 예외가 발생했으므로 `ExceptionResolver`가 작동한다. 가장 우선순위가 높은 `ExceptionHandlerExceptionResolver`가 실행된다.
* `ExceptionHandlerExceptionResolver`는 해당 컨트롤러에 `IllegalArgumentException`을 처리할 수 있는 `@ExceptionHandler`를 확인한다.
* `illegalExHandle()`를 실행한다. `@RestController`이므로 HTTP 컨버터가 사용되고, JSON 형태로 변환된다.
  * `ResponseEntity` 를 사용해서 HTTP 메시지 바디에 직접 응답한다. `ResponseEntity` 를 사용하면 HTTP 응답 코드를 프로그래밍해서 동적으로 변경할 수 있다.
* `@ResponseStatus(HttpStatus.Bad_REQUEST)`를 지정했으므로 HTTP 상태 코드 400으로 응답한다.



## API 예외 처리 - @ControllerAdvice

`@ExceptionHandler` 를 사용해서 예외를 깔끔하게 처리할 수 있게 되었다.
<u>하지만 정상 코드와 예외 처리 코드가 하나의 컨트롤러에 섞여 있다.</u> 
`@ControllerAdvice` 또는 `@RestControllerAdvice` 를 사용하면 둘을 분리할 수 있다.



### 예제 - 예외처리와 컨트롤러 분리

[`ExControllerAdvice`]

```java
package hello.exception.exhandler.advice;

import hello.exception.exception.UserException;
import hello.exception.exhandler.ErrorResult;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@Slf4j
@RestControllerAdvice
public class ExControllerAdvice {

    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(IllegalArgumentException.class)
    public ErrorResult illegalExHandle(IllegalArgumentException e) {
        log.error("[exceptionHandle] ex", e);
        return new ErrorResult("BAD", e.getMessage());
    }

    @ExceptionHandler
    public ResponseEntity<ErrorResult> userExHandle(UserException e) {
        log.error("[exceptionHandle] ex", e);
        ErrorResult errorResult = new ErrorResult("USER-EX", e.getMessage());
        return new ResponseEntity<>(errorResult, HttpStatus.BAD_REQUEST);
    }

    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    @ExceptionHandler
    public ErrorResult exHandle(Exception e) {
        log.error("[exceptionHandle] ex", e);
        return new ErrorResult("EX", "내부 오류");
    }

}
```



### @ControllerAdvice

* `@ControllerAdvice`는 대상으로 지정한 여러 컨트롤러에 `@ExceptionHandler` , `@InitBinder` 기능을 부여해주는 역할을 한다.
* ` @ControllerAdvice`에 대상을 지정하지 않으면 모든 컨트롤러에 적용된다. (글로벌 적용)
* `@RestControllerAdvice` 는 `@ControllerAdvice` 와 같고, `@ResponseBody` 가 추가되어 있다.



> `@ControllerAdvice` 
>
> 스프링 프레임워크에서 <u>전역적으로 적용되는 컨트롤러에 대한 설정과 도움말을 제공하는 어노테이션</u>이다. 
> 이 어노테이션을 사용하면 여러 컨트롤러에서 공통적으로 적용되는 설정을 중앙에서 관리할 수 있다.



> `@InitBinder`
>
> 스프링 프레임워크에서 사용되는 어노테이션 중 하나로, 웹 요청의 데이터를 <u>컨트롤러의 메서드에 전달하기 전에 데이터를 바인딩하거나 검증하기 위해 사용된다.</u>
>
> 주로 폼 데이터나 요청 파라미터를 자바 객체로 변환할 때 사용된다. 이 어노테이션을 메서드에 적용하면, <u>해당 메서드는 컨트롤러에 정의된 모든 핸들러 메서드보다 먼저 호출되어 데이터 바인딩이나 검증을 수행할 수 있다.</u>



#### 대상 컨트롤러 지정 방법

아래와 같은 다양한 대상을 지정할 수 있다.

* 전역 설정
* 특정 어노테이션 설정
* 패키지 설정
* 특정 클래스 설정

```java
// Target all Controllers annotated with @RestController
@ControllerAdvice(annotations = RestController.class)
public class ExampleAdvice1 {}

// Target all Controllers within specific packages
@ControllerAdvice("org.example.controllers")
public class ExampleAdvice2 {}

// Target all Controllers assignable to specific classes
@ControllerAdvice(assignableTypes = {ControllerInterface.class,
AbstractController.class})
public class ExampleAdvice3 {}
```



### 정리

`@ExceptionHandler` 와 `@ControllerAdvice` 를 조합하면 예외를 깔끔하게 해결할 수 있다.
