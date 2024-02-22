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









