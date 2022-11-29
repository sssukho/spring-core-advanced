# Section 2. ThreadLocal

## 필드 동기화 - 개발

앞서 로그 추적기를 만들면서 다음 로그를 출력할 때 트랜잭션ID 와 level 을 동기화 하는 문제가 있었다.

이 문제를 해결하기 위해 TraceId 를 파라미터로 넘기도록 구현했다. 이렇게 해서 동기화는 성공했지만, 로그를 출력하는 모든 메서드에 TraceId 파라밑커를 추가해야 하는 문제가 발생했다. TraceId 를 파라미터로 넘기지 않고 이 문제를 해결할 수 있는 방법은 없을까?

이런 문제를 해결할 목적으로 새로운 로그 추적기를 만들어보자. 이제 프로토타입 버전이 아닌 정식 버전으로 제대로 개발해보자. 향후 다양한 구현체로 변경할 수 있도록 `LogTrace` 인터페이스를 먼저 만들어 구현해보자.



### LogTrace 인터페이스

``` java
package hello.advanced.trace.logtrace;

import hello.advanced.trace.TraceStatus;

public interface LogTrace {
    TraceStatus begin(String message);

    void end(TraceStatus status);

    void exception(TraceStatus status, Exception e);
}
```

`LogTrace` 인터페이스에는 로그 추적기를 위한 최소한의 기능인 begin(), end(), exception() 을 정의했다.

이제 파라미터를 넘기지 않고 TraceId 를 동기화 할 수 있는 FieldLogTrace 구현체를 만들어보자.



### FiledLogTrace

``` java
package hello.advanced.trace.logtrace;

import hello.advanced.trace.TraceId;
import hello.advanced.trace.TraceStatus;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class FieldLogTrace implements LogTrace{

    private static final String START_PREFIX = "-->";
    private static final String COMPLETE_PREFIX = "<--";
    private static final String EX_PREFIX = "<X-";

    private TraceId traceIdHolder; // traceId 동기화, 동시성 이슈 발생

    @Override
    public TraceStatus begin(String message) {
        syncTraceId();
        TraceId traceId = traceIdHolder;
        Long startTimeMs = System.currentTimeMillis();
        log.info("[{}] {}{}", traceId.getId(), addSpace(START_PREFIX,
                traceId.getLevel()), message);

        return new TraceStatus(traceId, startTimeMs, message);
    }

    @Override
    public void end(TraceStatus status) {
        complete(status, null);
    }

    @Override
    public void exception(TraceStatus status, Exception e) {
        complete(status, e);
    }

    private void complete(TraceStatus status, Exception e) {
        Long stopTimeMs = System.currentTimeMillis();
        long resultTimeMs = stopTimeMs - status.getStartTimeMs();
        TraceId traceId = status.getTraceId();
        if (e == null) {
            log.info("[{}] {}{} time={}ms", traceId.getId(),
                    addSpace(COMPLETE_PREFIX, traceId.getLevel()), status.getMessage(),
                    resultTimeMs);
        } else {
            log.info("[{}] {}{} time={}ms ex={}", traceId.getId(),
                    addSpace(EX_PREFIX, traceId.getLevel()), status.getMessage(), resultTimeMs,
                    e.toString());
        }

        releaseTraceId();
    }

    private void syncTraceId() {
        if (traceIdHolder == null) {
            traceIdHolder = new TraceId();
        } else {
            traceIdHolder = traceIdHolder.createNextId();
        }
    }

    private void releaseTraceId() {
        if (traceIdHolder.isFirstLevel()) {
            traceIdHolder = null; // destroy
        } else {
            traceIdHolder = traceIdHolder.createPreviousId();
        }
    }

    private static String addSpace(String prefix, int level) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < level; i++) {
            sb.append( (i == level - 1) ? "|" + prefix : "|   ");
        }
        return sb.toString();
    }
}
```

FieldLogTrace 는 기존에 만들었던 `HelloTraceV2` 와 거의 같은 기능을 한다. `TraceId` 를 동기화 하는 부분만 파라미터를 사용하는 것에서 `TraceId traceIdHolder` 필드를 사용하도록 변경되었다. 이제 직전 로그의 TraceId 는 파라미터로 전달되는 것이 아니라 `FieldLogTrace` 의 필드인 `traceIdHolder` 에 저장된다.

여기서 중요한 부분은 로그를 시작할 때 호출하는 `syncTraceId()` 와 로그를 종료할 때 호출하는 `releaseTraceId()` 이다.

- syncTraceId()
  - TraceId 를 새로 만들거나 앞선 로그의 TraceId 를 참고해서 동기화하고, level 도 증가한다.
  - 최초 호출이면 TraceId 를 새로 만든다.
  - 직전 로그가 있으면 해당 로그의 TraceId 를 참고해서 동기화하고, level 도 하나 증가한다.
  - 결과를 traceIdHolder에 보관한다.
- releaseTraceId()
  - 메서드를 추가로 호출할 때는 level 이 하나 증가해야 하지만, 메서드 호출이 끝나면 level이 하나 감소해야 한다.
  - releaseTraceId() 는 level을 하나 감소한다.
  - 만약 최초 호출(level == 0) 이면 내부에서 관리하는 traceId를 제거한다.

```
[c80f5dbb] OrderController.request() //syncTraceId(): 최초 호출 level=0 
[c80f5dbb] |-->OrderService.orderItem() //syncTraceId(): 직전 로그 있음 level=1 증가
[c80f5dbb] | |-->OrderRepository.save() //syncTraceId(): 직전 로그 있음 level=2 증가
[c80f5dbb] | |<--OrderRepository.save() time=1005ms // releaseTraceId():level=2->1 감소
[c80f5dbb] |<--OrderService.orderItem() time=1014ms // level=1->0 감소
[c80f5dbb] OrderController.request() time=1017ms    // level==0, traceId 제거
```



### FieldLogTraceTest

``` java
package hello.advanced.trace.logtrace;

import static org.junit.jupiter.api.Assertions.*;

import hello.advanced.trace.TraceStatus;
import org.junit.jupiter.api.Test;

class FieldLogTraceTest {

    FieldLogTrace trace = new FieldLogTrace();

    @Test
    void begin_end_level2() {
        TraceStatus status1 = trace.begin("hello1");
        TraceStatus status2 = trace.begin("hello2");
        trace.end(status2);
        trace.end(status1);
    }

    @Test
    void begin_exception_level2() {
        TraceStatus status1 = trace.begin("hello");
        TraceStatus status2 = trace.begin("hello2");
        trace.exception(status2, new IllegalStateException());
        trace.exception(status1, new IllegalStateException());
    }

}
```

실행 결과를 보면 트랜잭션ID 도 동일하게 나오고, level을 통한 깊이도 잘 표현된다. FieldLogTrace.traceIdHolder 필드를 사용해서 TraceId 가 잘 동기화 되는 것을 확인할 수 있다.

이제 불필요하게 TraceId 를 파라미터로 전달하지 않아도 되고, 애플리케이션의 메서드 파라미터도 변경하지 않아도 된다.



## 필드 동기화 - 적용

지금까지 만든 FieldLogTrace를 애플리케이션에 적용해보자.



### LogTrace 스프링 빈 등록

FieldLogTrace 를 수동으로 스프링 빈으로 등록하자. 수동으로 등록하면 향후 구현체를 편리하게 변경할 수 있다는 장점이 있다.

``` java
package hello.advanced;

import hello.advanced.trace.logtrace.FieldLogTrace;
import hello.advanced.trace.logtrace.LogTrace;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class LogTraceConfig {

    @Bean
    public LogTrace logTrace() {
        return new FieldLogTrace();
    }
}
```



### v2 -> v3 복사

로그 추적기 V3를 적용하기 전에 먼저 기존 코드를 복사하자. (소스코드 참고)

- OrderControllerV3

  ``` java
  package hello.advanced.app.v3;
  
  import hello.advanced.trace.TraceStatus;
  import hello.advanced.trace.hellotrace.HelloTraceV2;
  import hello.advanced.trace.logtrace.LogTrace;
  import lombok.RequiredArgsConstructor;
  import org.springframework.web.bind.annotation.GetMapping;
  import org.springframework.web.bind.annotation.RestController;
  
  @RestController
  @RequiredArgsConstructor
  public class OrderControllerV3 {
  
      private final OrderServiceV3 orderService;
      private final LogTrace trace;
  
      @GetMapping("/v3/request")
      public String request(String itemId) {
          TraceStatus status = null;
          try {
              status = trace.begin("OrderController.request()");
              orderService.orderItem(status.getTraceId(), itemId);
              trace.end(status);
              return "ok";
          } catch (Exception e) {
              trace.exception(status, e);
              throw e;
          }
      }
  }
  ```

- OrderServiceV3

  ``` java
  package hello.advanced.app.v3;
  
  import hello.advanced.trace.TraceId;
  import hello.advanced.trace.TraceStatus;
  import hello.advanced.trace.hellotrace.HelloTraceV2;
  import hello.advanced.trace.logtrace.LogTrace;
  import lombok.RequiredArgsConstructor;
  import org.springframework.stereotype.Service;
  
  @Service
  @RequiredArgsConstructor
  public class OrderServiceV3 {
  
      private final OrderRepositoryV3 orderRepository;
      private final LogTrace trace;
  
      public void orderItem(TraceId traceId, String itemId) {
          TraceStatus status = null;
          try {
              status = trace.begin("OrderService.orderItem()");
              orderRepository.save(status.getTraceId(), itemId);
              trace.end(status);
          } catch (Exception e) {
              trace.exception(status, e);
              throw e;
          }
      }
  }
  ```

- OrderRepositoryV3

  ``` java
  package hello.advanced.app.v3;
  
  import hello.advanced.trace.TraceId;
  import hello.advanced.trace.TraceStatus;
  import hello.advanced.trace.hellotrace.HelloTraceV2;
  import hello.advanced.trace.logtrace.LogTrace;
  import lombok.RequiredArgsConstructor;
  import org.springframework.stereotype.Repository;
  
  @Repository
  @RequiredArgsConstructor
  public class OrderRepositoryV3 {
  
      private final LogTrace trace;
  
      public void save(TraceId traceId, String itemId) {
  
          TraceStatus status = null;
          try {
              status = trace.begin("OrderService.orderItem()");
              // 저장 로직
              if (itemId.equals("ex")) {
                  throw new IllegalStateException("예외 발생!");
              }
              sleep(1000);
              trace.end(status);
          } catch (Exception e) {
              trace.exception(status, e);
              throw e;
          }
      }
  
      private void sleep(int millis) {
          try {
              Thread.sleep(millis);
          } catch (InterruptedException e) {
              e.printStackTrace();
          }
      }
  }
  ```



## 필드 동기화 - 동시성 문제

### 동시성 문제 확인

다음 로직을 1초 안에 2번 호출해보자.

- http://localhost:8080/v3/request?itemId=hello

- http://localhost:8080/v3/request?itemId=hello

- 기대결과

  ```
  [nio-8080-exec-3] [52808e46] OrderController.request()
  [nio-8080-exec-3] [52808e46] |-->OrderService.orderItem()
  [nio-8080-exec-3] [52808e46] |   |-->OrderRepository.save()
  [nio-8080-exec-4] [4568423c] OrderController.request()
  [nio-8080-exec-4] [4568423c] |-->OrderService.orderItem()
  [nio-8080-exec-4] [4568423c] |   |-->OrderRepository.save()
  [nio-8080-exec-3] [52808e46] |   |<--OrderRepository.save() time=1001ms
  [nio-8080-exec-3] [52808e46] |<--OrderService.orderItem() time=1001ms
  [nio-8080-exec-3] [52808e46] OrderController.request() time=1003ms
  [nio-8080-exec-4] [4568423c] |   |<--OrderRepository.save() time=1000ms
  [nio-8080-exec-4] [4568423c] |<--OrderService.orderItem() time=1001ms
  [nio-8080-exec-4] [4568423c] OrderController.request() time=1001ms
  ```

- 실제 결과

  ```
  [nio-8080-exec-1] [cb3fdfb0] OrderController.request()
  [nio-8080-exec-1] [cb3fdfb0] |-->OrderService.orderItem()
  [nio-8080-exec-1] [cb3fdfb0] |   |-->OrderService.orderItem()
  [nio-8080-exec-2] [cb3fdfb0] |   |   |-->OrderController.request()
  [nio-8080-exec-2] [cb3fdfb0] |   |   |   |-->OrderService.orderItem()
  [nio-8080-exec-2] [cb3fdfb0] |   |   |   |   |-->OrderService.orderItem()
  [nio-8080-exec-1] [cb3fdfb0] |   |<--OrderService.orderItem() time=1000ms
  [nio-8080-exec-1] [cb3fdfb0] |<--OrderService.orderItem() time=1003ms
  [nio-8080-exec-1] [cb3fdfb0] OrderController.request() time=1003ms
  [nio-8080-exec-2] [cb3fdfb0] |   |   |   |   |<--OrderService.orderItem() time=1004ms
  [nio-8080-exec-2] [cb3fdfb0] |   |   |   |<--OrderService.orderItem() time=1005ms
  [nio-8080-exec-2] [cb3fdfb0] |   |   |<--OrderController.request() time=1005ms
  ```

- 기대한 것과는 다르게 트랜잭션ID 도 동일하고, level도 뭔가 많이 꼬인듯이 로그가 찍힌다. 분명히 테스트 코드로 작성할 때는 문제가 없었는데 무엇이 문제일까?



### 동시성 문제

사실 이 문제는 동시성 문제이다.

`FieldLogTrace` 는 싱글톤으로 등록된 스프링 빈이다. 이 객체의 인스턴스가 애플리케이션에 딱 1개 존재한다는 뜻이다. 이렇게 하나만 있는 인스턴스의 FieldlogTrace.traceIdHolder 필드를 여러 쓰레드가 동시에 접근하기 때문에 문제가 발생한다.













