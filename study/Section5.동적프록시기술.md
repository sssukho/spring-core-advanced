# Section 5. 동적 프록시 기술

## 리플렉션

지금까지 프록시를 사용해서 기존 코드를 변경하지 않고, 로그 추적기라는 부가 기능을 적용할 수 있었다. 그런데 문제는 대상 클래스 수 만큼 로그 추적을 위한 프록시 클래스를 만들어야 한다는 점이다. 로그 추적을 위한 프록시 클래스들의 소스코드는 거의 같은 모양을 하고 있다.

자바가 기본으로 제공하는 JDK 동적 프록시 기술이나 CGLIB 같은 프록시 생성 오픈소스 기술을 활용하면 프록시 객체를 동적으로 만들어낼 수 있다. 쉽게 이야기해서 프록시 클래스를 지금처럼 계속 만들지 않아도 된다는 것이다. 프록시를 적용할 코드를 하나만 만들어두고 동적 프록시 기술을 사용해서 프록시 객체를 찍어내면 된다.

JDK 동적 프록시를 이해하기 위해서는 먼저 자바의 리플렉션 기술을 이해해야 한다. 리플렉션 기술을 사용하면 클래스나 메서드의 메타정보를 동적으로 획득하고, 코드도 동적으로 호출할 수 있다.

여기서는 JDK 동적 프록시를 이해하기 위한 최소한의 리플랙션 기술을 알아보자.



#### ReflectionTest

``` java
package hello.proxy.jdkdynamic;

import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;

@Slf4j
public class ReflectionTest {

    @Test
    void reflection0() {
        Hello target = new Hello();
        
        // 공통 로직1 시작
        log.info("start");
        String result1 = target.callA(); // 호출하는 메서드가 다름
        log.info("result={}", result1);
        // 공통 로직1 종료
        
        // 공통 로직2 시작
        log.info("start");
        String result2 = target.callB(); // 호출하는 메서드가 다름
        log.info("result={}", result2);
        // 공통 로직2 종료
    }
    
    @Slf4j
    static class Hello {
        public String callA() {
            log.info("callA");
            return "A";
        }
        
        public String callB() {
            log.info("callB");
            return "B";
        }
    }
}
```

- 공통 로직1과 공통 로직2는 호출하는 메서드만 다르고 전체 코드 흐름이 완전히 같다.
  - 먼저 start 로그를 출력한다.
  - 어떤 메서드를 호출한다.
  - 메서드의 호출 결과를 로그로 출력한다.
- 여기서 공통 로직1과 공통 로직2를 하나의 메서드로 뽑아서 합칠 수 있을까?
- 쉬워 보이지만 메서드로 뽑아서 공통화하는 것이 생각보다 어렵다. 왜냐하면 중간에 호출하는 메서드가 다르기 때문이다.
- 호출하는 메서드인 `target.callA()`, `target.callB()` 이 부분만 동적으로 처리할 수 있다면 문제를 해결할 수 있을 듯 하다.

``` java
log.info("start");
String result = xxx(); // 호출 대상이 다름, 동적 처리 필요
log.info("result={}", result);
```

이럴 때 사용하는 기술이 바로 리플렉션이다. 리플렉션은 클래스나 메서드의 메타정보를 사용해서 동적으로 호출하는 메서드를 변경할 수 있다. 바로 리플렉션을 사용해보자.

> [참고]
>
> 람다를 사용해서 공통화 하는 것도 가능하다. 여기서는 람다를 사용하기 어려운 상황이라 가정하자. 그리고 리플렉션 학습이 목적이니 리플렉션에 집중하자.



#### ReflectionTest - reflection1 추가

``` java
@Test
void reflection1() throws Exception {
  // 클래스 정보
  Class classHello = Class.forName("hello.proxy.jdkdynamic.ReflectionTest$Hello");

  Hello target = new Hello();

  // callA 메서드 정보
  Method methodCallA = classHello.getMethod("callA");
  Object result1 = methodCallA.invoke(target);
  log.info("result1={}", result1);

  // callB 메서드 정보
  Method methodCallB = classHello.getMethod("callB");
  Object result2 = methodCallB.invoke(target);
  log.info("result2={}", result2);
}
```

- `Class.forName("hello.proxy.jdkdynamic.ReflectionTest$Hello")` : 클래스 메타정보를 획득한다. 참고로 내부 클래스는 구분을 위해 $ 를 사용한다.
- `classHello.getMethod("call")`: 해당 클래스의 call 메서드 메타정보를 획득한다.
- `methodCallA.invoke(target)`: 획득한 메서드 메타정보로 실제 인스턴스의 메서드를 호출한다. 여기서 methodCallA 는 Hello 클래스의 callA() 라는 메서드 메타정보다. methodCallA.invoke(인스턴스) 를 호출하면서 인스턴스를 넘겨주면 해당 인스턴스의 callA() 메서드를 찾아서 실행한다. 여기서는 target의 callA() 메서드를 호출한다.

그런데 target.callA() 나 target.callB() 메서드를 직접 호출하면 되지 이렇게 메서드 정보를 획득해서 메서드를 호출하면 어떤 효과가 있을까? 여기서 중요한 핵심은 클래스나 메서드 정보를 동적으로 변경할 수 있다는 점이다.

기존의 callA(), callB() 메서드를 직접 호출하는 부분이 Method 로 대체되었다. 덕분에 이제 공통 로직을 만들 수 있게 되었다.



#### ReflectionTest - reflection2 추가

``` java
@Test
void reflection2() throws Exception {
  // 클래스 정보
  Class classHello = Class.forName("hello.proxy.jdkdynamic.ReflectionTest$Hello");

  Hello target = new Hello();

  Method methodCallA = classHello.getMethod("callA");
  dynamicCall(methodCallA, target);

  // callB 메서드 정보
  Method methodCallB = classHello.getMethod("callB");
  dynamicCall(methodCallB, target);
}

private void dynamicCall(Method method, Object target) throws Exception {
  log.info("start");
  Object result = method.invoke(target);
  log.info("result={}", result);
}
```

- `dynamicCall(Method method, Object target)`
  - 공통 로직1, 공통 로직2 를 한번에 처리할 수 있는 통합된 공통 처리 로직이다.
  - `Method method`: 첫번째 파라미터는 호출할 메서드 정보가 넘어온다. 이것이 핵심이다. 기존에는 메서드 이름을 직접 호출했지만, 이제는 Method 라는 메타정보를 통해서 호출할 메서드 정보가 동적으로 제공된다.
  - Object target : 실제 실행할 인스턴스 정보가 넘어온다. 타입이 `Object` 라는 것은 어떠한 인스턴스도 받을 수 있다는 뜻이다. 물론 `method.invoke(target)` 를 사용할 때 호출할 클래스와 메서드 정보가 서로 다르면 예외가 발생한다.



### 정리

정적인 `target.call()`, `target.callB()` 코드를 리플렉션을 사용해서 `Method` 라는 메타정보로 추상화했다. 덕분에 공통 로직을 만들 수 있게 되었다.



### 주의

리플렉션을 사용하면 클래스와 메서드의 메타정보를 사용해서 애플리케이션을 동적으로 유연하게 만들 수 있다. 하지만 리플렉션 기술은 런타임에 동작하기 때문에 컴파일 시점에 오류를 잡을 수 없다.

예를 들어서 지금까지 살펴본 코드에서 `getMethod("callA")` 안에 들어가는 문자를 실수로 `getMethod("callZ")` 로 작성해도 컴파일 오류가 발생하지 않는다. 그러나 해당 코드를 직접 실행하는 시점에 발생하는 오류인 런타임 오류가 발생한다.

가장 좋은 오류는 개발자가 즉시 확인할 수 있는 컴파일 오류이고, 가장 무서운 오류는 사용자가 직접 실행할 때 발생하는 런타임 오류다.

따라서 리플렉션은 일반적으로 사용하면 안된다. 지금까지 프로그래밍 언어가 발달하면서 타입 정보를 기반으로 컴파일 시점에 오류를 잡아준 덕분에 개발자가 편하게 살았는데, 리플렉션은 그것에 역행하는 방식이다.

리플렉션은 프레임워크 개발이나 또는 매우 일반적인 공통 처리가 필요할 때 부분적으로 주의해서 사용해야 한다.



## JDK 동적 프록시 - 소개

지금까지 프록시를 적용하기 위해 적용 대상의 숫자 만큼 많은 프록시 클래스를 만들었다. 적용 대상이 100개면 프록시 클래스도 100개 만들었다. 그런데 앞서 살펴본 것 과 같이 프록시 클래스의 기본 코드와 흐름은 거의 같고, 프록시를 어떤 대상에 적용하는가 정도만 차이가 있었다. 쉽게 이야기해서 프록시의 로직은 같은데, 적용 대상만 차이가 있는 것이다.

이 문제를 해결하는 것이 바로 동적 프록시 기술이다.

동적 프록시 기술을 사용하면 개발자가 직접 프록시 클래스를 만들지 않아도 된다. 이름 그대로 프록시 객체를 동적으로 런타임에 개발자 대신 만들어준다. 그리고 동적 프록시에 원하는 실행 로직을 지정할 수 있다.

> [주의]
>
> JDK 동적 프록시는 인터페이스를 기반으로 프록시를 동적으로 만들어준다. 따라서 인터페이스가 필수이다.



### 기본 예제 코드

간단히 A,B 클래스를 만드는데, JDK 동적 프록시는 인터페이스가 필수이다. 따라서 인터페이스와 구현체로 구분했다.



#### AInterface (테스트 코드 하위)

``` java
package hello.proxy.jdkdynamic.code;

public interface AInterface {
    String call();
}
```



#### AImpl (테스트 코드 하위)

``` java
package hello.proxy.jdkdynamic.code;

import lombok.extern.slf4j.Slf4j;

@Slf4j
public class AImpl implements AInterface{

    @Override
    public String call() {
        log.info("A 호출");
        return "a";
    }
}
```



#### BInterface (테스트 코드 하위)

``` java
package hello.proxy.jdkdynamic.code;

public interface BInterface {
    String call();
}
```



#### BImpl (테스트 코드 하위)

``` java
package hello.proxy.jdkdynamic.code;

import lombok.extern.slf4j.Slf4j;

@Slf4j
public class BImpl implements BInterface {

    @Override
    public String call() {
        log.info("B 호출");
        return "b";
    }
}
```



## JDK 동적 프록시 - 예제 코드

### JDK 동적 프록시 InvocationHandler

JDK 동적 프록시에 적용할 로직은 `InvocationHandler` 인터페이스를 구현해서 작성하면 된다.



#### JDK 동적 프록시가 제공하는 InvocationHandler

``` java
public interface InvocationHandler {
  
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
}
```

- `Object proxy` : 프록시 자신
- `Method method` : 호출한 메서드
- `Object[] args` : 메서드를 호출할 떄 전달한 인수



#### TimeInvocationHandler (테스트 코드 하위)

``` java
package hello.proxy.jdkdynamic.code;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class TimeInvocationHandler implements InvocationHandler {

    private final Object target;

    public TimeInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object o, Method method, Object[] args) throws Throwable {
        log.info("TimeProxy 실행");
        long startTime = System.currentTimeMillis();

        Object result = method.invoke(target, args);
        
        long endTime = System.currentTimeMillis();
        long resultTime = endTime - startTime;
        log.info("TimeProxy 종료 resultTime={}", resultTime);
        return result;
    }
}
```

- `TimeInvocationHandler` 은 `InvocationHandler` 인터페이스를 구현한다. 이렇게해서 JDK 동적 프록시에 적용할 공통 로직을 개발할 수 있다.
- `Object target` : 동적 프록시가 호출할 대상
- `method.invoke(target, args)` : 리플렉션을 사용해서 traget 인스턴스의 메서드를 실행한다. `args` 는 메서드 호출시 넘겨줄 인수이다.

이제 테스트 코드로 JDK 동적 프록시를 사용해보자.



#### JdkDynamicProxyTest

``` java
package hello.proxy.jdkdynamic;

import hello.proxy.jdkdynamic.code.AImpl;
import hello.proxy.jdkdynamic.code.AInterface;
import hello.proxy.jdkdynamic.code.BImpl;
import hello.proxy.jdkdynamic.code.BInterface;
import hello.proxy.jdkdynamic.code.TimeInvocationHandler;
import java.lang.reflect.Proxy;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;

@Slf4j
public class JdkDynamicProxyTest {

    @Test
    void dynamicA() {
        AInterface target = new AImpl();
        TimeInvocationHandler handler = new TimeInvocationHandler(target);
        AInterface proxy = (AInterface) Proxy
                .newProxyInstance(AInterface.class.getClassLoader(), new Class[]{AInterface.class}, handler);
        proxy.call();
        log.info("targetClass={}", target.getClass());
        log.info("proxyClass={}", proxy.getClass());
    }
    
    @Test
    void dynamicB() {
        BInterface target = new BImpl();
        TimeInvocationHandler handler = new TimeInvocationHandler(target);
        BInterface proxy = (BInterface) Proxy
                .newProxyInstance(BInterface.class.getClassLoader(), new Class[]{BInterface.class}, handler);
        proxy.call();

        log.info("targetClass={}", target.getClass());
        log.info("proxyClass={}", proxy.getClass());
    }
}
```

- `new TimeInvocationHandler(target)` : 동적 프록시에 적용할 핸들러 로직이다.
- `(AInterface) Proxy.newProxyInstance(AInterface.class.getClassLoader(), new Class[]` 
  - 동적 프록시는 `java.lang.reflect.proxy` 를 통해서 생성할 수 있다.
  - 클래스 로더 정보, 인터페이스, 그리고 핸들러 로직을 넣어주면 된다. 그러면 해당 인터페이스를 기반으로 동적 프록시를 생성하고 그 결과를 반환한다.

```
## dynamicA() 출력 결과

TimeInvocationHandler - TimeProxy 실행
AImpl - A 호출
TimeInvocationHandler - TimeProxy 종료 resultTime=0
JdkDynamicProxyTest - targetClass=class hello.proxy.jdkdynamic.code.AImpl
JdkDynamicProxyTest - proxyClass=class com.sun.proxy.$Proxy9
```

출력 결과를 보면 프록시가 정상 수행된 것을 확인할 수 있다.



#### 생성된 JDK 동적 프록시

`proxyClass=class com.sun.proxy.$Proxy9` 이 부분이 동적으로 생성된 프록시 클래스 정보이다. 이것은 우리가 만든 클래스가 아니라 JDK 동적 프록시가 이름 그대로 동적으로 만들어준 프록시다. 이 프록시는 `TimeInvocationHandler` 로직을 실행한다.

#### 실행 순서

1. 클라이언트는 JDK 동적 프록시의 `call()` 을 실행한다.
2. JDK 동적 프록시는 `InvocationHandler.invoke()` 를 호출한다. `TimeInvocationHandler` 가 구현체로 있으므로 `TimeInvocationHandler.invoke()` 가 호출된다.
3. `TimeInvocationHandler` 가 내부 로직을 수행하고, `method.invoke(target, args)` 를 호출해서 `target` 인 실제 객체(`AImpl`) 를 호출한다.
4. `AImpl` 인스턴스의 `call()` 이 실행된다.
5. `AImpl` 인스턴스의 `call()` 의 실행이 끝나면 `TimeInvocationHandler` 로 응답이 돌아온다. 시간 로그를 출력하고 결과를 반환한다.

![5-1](./img/5-1.png)



#### 동적 프록시 클래스 정보

`dynamicA()` 와 `dynamicB()` 둘을 동시에 함께 실행하면 JDK 동적 프록시가 각각 다른 동적 프록시 클래스를 만들어주는 것을 확인할 수 있다.

```
proxyClass=class com.sun.proxy.$Proxy1 // dynamicA
proxyClass=class com.sun.proxy.$Proxy2 // dynamicB
```



#### 정리

예제를 보면 `Aimpl`, `BImpl` 각각 프록시를 만들지 않았다. 프록시는 JDK 동적 프록시를 사용해서 동적으로 만들고 `TimeInvocationHandler` 는 공통으로 사용했다.

JDK 동적 프록시 기술 덕분에 적용 대상 만큼 프록시 객체를 만들지 않아도 된다. 그리고 같은 부가 기능 로직을 한번만 개발해서 공통으로 적용할 수 있다. 만약 적용 대상이 100개 여도 동적 프록시를 통해서 생성하고, 각각 필요한 InvocationHandler 만 만들어서 넣어주면 된다.

결과적으로 프록시 클래스를 수 없이 만들어야 하는 문제도 해결하고, 부가 기능 로직도 하나의 클래스에 모아서 단일 책임 원칙(SRP)도 지킬 수 있게 되었다.

JDK 동적 프록시 없이 직접 프록시를 만들어서 사용할 때와 JDK 동적 프록시를 사용할 때의 차이를 그림으로 비교해보자.

![5-2](./img/5-2.png)

![5-3](./img/5-3.png)













