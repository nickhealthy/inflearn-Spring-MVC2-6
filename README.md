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

    //RequestDispatcher 상수로 정의되어 있음
    // 예외
    public static final String ERROR_EXCEPTION = "javax.servlet.error.exception";
    // 예외 타입
    public static final String ERROR_EXCEPTION_TYPE = "javax.servlet.error.exception_type";
    // 오류 메시지
    public static final String ERROR_MESSAGE = "javax.servlet.error.message";
    // 클라이언트 요청 URI
    public static final String ERROR_REQUEST_URI = "javax.servlet.error.request_uri";
    // 오류가 발생한 서블릿 이름
    public static final String ERROR_SERVLET_NAME = "javax.servlet.error.servlet_name";
    // HTTP 상태코드
    public static final String ERROR_STATUS_CODE = "javax.servlet.error.status_code";

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

