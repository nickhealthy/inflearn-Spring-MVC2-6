# 인프런 강의

해당 저장소의 `README.md`는 인프런 김영한님의 SpringBoot 강의 시리즈를 듣고 Spring 프레임워크의 방대한 기술들을 복기하고자 공부한 내용을 가볍게 정리한 것입니다.

문제가 될 시 삭제하겠습니다.



## 해당 프로젝트에서 배우는 내용

* 섹션 8 | 예외 처리와 오류 페이지



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



