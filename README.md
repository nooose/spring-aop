# Spring Framework AOP

## 프록시의 주요 기능
* 접근 제어 (프록시 패턴)
    * 권한에 따른 접근 차단
    * 캐싱
    * 지연 로딩
* 부가 기능 추가 (데코레이터 패턴)
    * 예) 요청 값이나, 응답 값을 중간에 변형
    * 예) 실행 시간을 측정해서 추가 로그를 남김
    
## 동적 프록시
### 리플렉션
클래스나 메서드의 메타정보를 동적으로 획득하고, 코드도 동적으로 호출할 수 있다.
* 리플렉션 예제
```java
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

    @Test
    void reflection() throws Exception {
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

    /*
    INFO hello.proxy.jdkdynamic.ReflectionTest$Hello - callA
    INFO hello.proxy.jdkdynamic.ReflectionTest - result1=A
    INFO hello.proxy.jdkdynamic.ReflectionTest$Hello - callB
    INFO hello.proxy.jdkdynamic.ReflectionTest - result2=B
    */
```

* 리플렉션을 통해 공통화
```java
    @Test
    void reflection() throws Exception {
        // 클래스 정보
        Class classHello = Class.forName("hello.proxy.jdkdynamic.ReflectionTest$Hello");

        Hello target = new Hello();
        // callA 메서드 정보
        Method methodCallA = classHello.getMethod("callA");
        dynamicCall(methodCallA, target);

        // callB 메서드 정보
        Method methodCallB = classHello.getMethod("callB");
        dynamicCall(methodCallA, target);
    }

    private void dynamicCall(Method method, Object target) throws InvocationTargetException, IllegalAccessException {
        log.info("start");
        Object result = method.invoke(target);
        log.info("result={}", result);
    }
```
> 리플렉션은 컴파일 시점에 오류를 잡지못하고 코드를 직접 실행하는 시점에 발생하는 런타임 오류가 발생하기 때문에 주의해야한다.

### JDK dynamic proxy
`InvocationHandler` 인터페이스를 구현해서 프록시에 적용할 공통 로직을 개발할 수 있다.
* 시간 측정을 하는 예제
```java
@Slf4j
public class TimeInvocationHandler implements InvocationHandler {
    private final Object target;

    public TimeInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
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
* `Object target`: 동적 프록시가 호출할 대상
* `args`: 메서드 호출시 넘겨줄 인수 

```java
    @Test
    void dynamicA() {
        AInterface target = new AImpl();
        TimeInvocationHandler handler = new TimeInvocationHandler(target);

        AInterface proxy = (AInterface) Proxy.newProxyInstance(AInterface.class.getClassLoader(), new Class[]{AInterface.class}, handler);

        proxy.call();
    }
```

### CGLIB (Code Generator Library)
* 바이트코드를 조작해서 동적으로 클래스를 생성하는 기술을 제공
* 인터페이스가 없어도 구체 클래스만 가지고 동적 프록시를 생성
스프링의 `ProxyFactory`가 편리하게 사용하게 도와주고 있다.

## 스프링 프록시 팩토리
인터페이스가 있으면 `JDK Dynamic Proxy`를 사용하고 구체 클래스가 있으면 `cglib`을 사용한다.

즉, 프록시 팩토리 하나로 편리하게 동적 프록시를 생성할 수 있다.

> Advice 도입
* 프록시가 호출하는 부가 기능.


* JDK dynamic proxy의 InvocationHandler
* cglib의 MethodInterceptor

위 두개 모두 Advice를 호출한다.
> Pointcut
* 어디에 부가 기능을 적용할지, 어디에 부가 기능을 적용하지 않을지 판단하는 필터링 로직.
특정 조건에 맞을 때 프록시 로직을 추가하는 경우에 사용된다.

    * `ClassFilter`: 클래스를 기준으로 필터링 
    * `MethodFilter`: 메서드를 기준으로 필터링

> Advisor
* `Pointcut` + `Advice`


### 사용 예제
Advice를 생성하고 ProxyFactory에 target과 생성한 Advice를 넣어주면 끝이다.
```java
// target 대상
@Slf4j
public class ServiceImpl implements ServiceInterface {

    @Override
    public void save() {
        log.info("save 호출");
    }

    @Override
    public void find() {
        log.info("find 호출");
    }
}


// Advice 생성
@Slf4j
public class TimeAdvice implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        // 공통 또는 중복 로직 시작
        log.info("TimeProxy 실행");
        long startTime = System.currentTimeMillis();
        
        // 비즈니스 로직 실행 부분
        Object result = invocation.proceed();

        long endTime = System.currentTimeMillis();
        long resultTime = endTime - startTime;
        log.info("TimeProxy 종료 resultTime={}", resultTime);

        // 공통 또는 중복 로직 종료
        return result;
    }
}
```
```java
// ProxyFactory 생성
@Test
void interfaceProxy() {
    ServiceInterface target = new ServiceImpl();
    ProxyFactory proxyFactory = new ProxyFactory(target);
    proxyFactory.addAdvice(new TimeAdvice());
    // 메서드 내부에서 Advisor가 생성된다.
    // DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(Pointcut.TRUE, new TimeAdvice());
    // proxyFactory.addAdvisor(advisor);

    ServiceInterface proxy = (ServiceInterface) proxyFactory.getProxy();

    log.info("targetClass={}", target.getClass());
    log.info("proxyClass={}", proxy.getClass());

    proxy.find();
    proxy.save();

    assertThat(AopUtils.isAopProxy(proxy)).isTrue();
    assertThat(AopUtils.isJdkDynamicProxy(proxy)).isTrue(); // 인터페이스가 있는 타겟이기 때문에 JdkDynamicProxy로 생성되었음을 확인 
    assertThat(AopUtils.isCglibProxy(proxy)).isFalse();
}

/* 실행결과
INFO hello.proxy.proxyfactory.ProxyFactoryTest - targetClass=class hello.proxy.common.service.ServiceImpl
INFO hello.proxy.proxyfactory.ProxyFactoryTest - proxyClass=class com.sun.proxy.$Proxy13
INFO hello.proxy.common.advice.TimeAdvice - TimeProxy 실행
INFO hello.proxy.common.service.ServiceImpl - find 호출
INFO hello.proxy.common.advice.TimeAdvice - TimeProxy 종료 resultTime=0
INFO hello.proxy.common.advice.TimeAdvice - TimeProxy 실행
INFO hello.proxy.common.service.ServiceImpl - save 호출
INFO hello.proxy.common.advice.TimeAdvice - TimeProxy 종료 resultTime=0
*/
```

### 빈 후처리기
* Bean으로 등록하기 위한 동적 프록시 생성코드가 많아지는 단점
* 실제 객체 대신 프록시 객체를 Bean으로 등록 해야하는 단점

위 두가지 문제를 해결한다.