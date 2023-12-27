---
title: 스프링 빈 스코프(Spring Bean Scope) 알아보기
categories: [Computer engineering, Backend engineering]
tags: [backend, spring, java, spring bean scope, 백엔드, 스프링, 자바, 스프링 빈 스코프]
---

이번 포스팅에서는 Spring Bean이 존재할 수 있는 범위를 뜻하는 Spring Bean Scope에 대해 자세히 알아보겠습니다.

## Spring Bean Scope란?
우선 **Spring Bean Scope는 해당 Bean이 존재할 수 있는 범위**를 뜻합니다. 이전의 포스팅들에서는 별다른 설명없이 Spring Bean이 Spring Container의 생성과 동시에 만들어져서 종료되기 직전에 소멸되는 것처럼 설명했습니다. 이는 반은 맞고 반은 틀린 설명입니다. 왜냐하면, 이건 Default scope인 싱글톤 스코프일 때만 해당되는 이야기이기 때문입니다.   
    
싱글톤 스코프 외에도 스프링에서는 다음과 같은 다양한 스코프를 지원하고 있습니다.   
    
1. singleton scope: Default 스코프, 스프링 컨테이너 생성과 동시에 생성, 스프링 컨테이너의 시작에서부터 종료될 때까지 유지되는 가장 넓은 범위의 스코프
2. prototype scope: 해당 Bean이 조회될 때 생성, 스프링 컨테이너가 Bean의 생성, 의존 관계 주입, 초기화 메서드 호출까지만 관여해서 해주고 그 이후에는 더 이상 관리하지 않는 매우 짧은 범위의 스코프. 그래서 종료 메서드 호출이 안됨.
3. request scope: 웹 request가 들어오고나서 response가 나갈 때까지만 유지되는 굉장히 특별한 스코프
4. session scope: 웹 session이 생성되고 종료될 때까지 유지되는 스코프
5. application scope: 웹의 서블릿 컨텍스트와 같은 범위로 유지되는 스코프   
    
이러한 Bean scope는 아래와 같이 등록할 수 있습니다.   
    
[컴포넌트 스캔 자동 등록을 사용할 때]
```java
@Scope("prototype")
@Component
public class HelloBean {}
```

[수동 등록을 사용할 때]
```java
@Scope("prototype")
@Bean
PrototypeBean HelloBean() {
    return new HelloBean();
}
```

## Prototype scope란?
**Default scope인 Singleton scope는 같은 타입에 대해서 1개의 인스턴스만을 유지**하기 때문에 **Singleton scope를 가지는 Spring Bean을 조회하면 항상 같은 인스턴스의 스프링 빈을 반환**합니다. 반면에 **Prototype scope를 가지는 Bean을 조회하면 스프링 컨테이너는 항상 새로운 인스턴스를 생성해서 반환**합니다. 

![1](/assets/img/spring-bean-scope/1.png){: w="500" h="300" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

위 그림은 **Singleton scope로 관리되는 Bean**에 대한 설명입니다. 그림처럼 동시에 요청이 들어오건 순차적으로 요청이 들어오건 관계 없이 **같은 타입의 Bean에 대해서는 동일한 인스턴스를 반환해주고, 스프링 컨테이너 생성 시점부터 종료시점까지 직접 Bean을 관리**합니다.

![2](/assets/img/spring-bean-scope/2.png){: w="500" h="300" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

반면에, **Prototype scope Bean 같은 경우에는 위 그림처럼 요청이 들어올 때마다 새로운 빈을 생성하고 의존 관계를 주입하고 초기화 메서드까지만 호출해준 다음에, 그대로 return하고 그 이후로는 스프링 컨테이너에서 지워버리고 관리해주지 않습니다.**   
    
**즉, Prototype Bean을 관리할 책임은 온전히 Bean을 전달받은 클라이언트에 있게 됩니다. 따라서, @PreDestroy와 같은 종료 메서드가 자동 호출되지 않으므로 주의**해야합니다.  

추가적으로 반드시 알아둬야 할 것이, **Singleton Bean은 스프링 컨테이너 생성 시점에 생성되며 이때 초기화 메서드가 실행**되지만, Prototype Bean은 스프링 컨테이너에서 빈을 조회하는 시점에 생성되고 초기화 메서드도 이때 실행됩니다.  
    
## Prototype Bean과 Singleton Bean을 함께 사용 했을 때 문제점
일반적으로 보통 Singleton Bean을 주로 사용하겠지만, **Prototype Bean을 활용하여 어떤 로직을 호출할 때마다 새롭게 Bean을 생성하고 싶을 때가 있을 수 있습니다. 그렇게 되면 두 스코프의 Bean을 함께 사용하게 되는데 이때 큰 문제가 발생할 수 있습니다.** 언제 문제가 발생하는지 아래 그림을 통해 알아보겠습니다.   
    
![3](/assets/img/spring-bean-scope/3.png){: w="500" h="300" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

위 그림을 보면 Singleton Bean이 하나 있고 그 Bean의 field로 Prototype Bean을 주입받고 있습니다. 이렇게 되면 그림 설명과 같이 **주입 받을 당시에는 Spring container로부터 Prototype Bean 생성을 요청해 새로 받는 것이 맞지만, 그 이후에는 Singleton Bean 자체적으로 reference를 가지고 계속 참조**하게 됩니다. 따라서, **일반적인 Prototype Bean처럼 Prototpye Bean의 로직을 호출할 때마다 새로 생성하길 원했다면, 원하는대로 동작하지 않게됩니다.**

![4](/assets/img/spring-bean-scope/4.png){: w="500" h="300" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

그 예시가 바로 위에 있습니다. Prototype Bean의 addCount() 메서드를 통해 count 값을 1씩 증가시켜주고 있는데, 새로 생성되지 않다보니 계속해서 동일한 Prototype Bean의 count 값에 누적합이 되는 문제가 발생합니다.

## Prototype Bean과 Singleton Bean을 함께 사용 했을 때 발생하는 문제 해결하기
방금 전에 개발자가 Prototype Bean을 활용하여 어떤 로직을 호출할 때마다 새롭게 Bean을 생성하여 사용하길 원했지만 의도한대로 구현되지 않는 문제를 봤습니다. 이 문제를 어떻게 해결할 수 있을까요?   
    
**조금 더럽지만, 가장 간단한 해결 방법은 Singleton Bean에 field로 ApplicationContext를 만들고 @Autowired하여 의존 관계를 주입받은 뒤 필요할 때 마다 getBean()하여 꺼내 쓰게되면 그때마다 스프링 컨테이너에 요청하여 새로운 Prototype Bean을 생성**받을 수 있습니다.   
    
**하지만 이 방법의 문제는 우리는 스프링 컨테이너에서 내가 필요한 Prototype Bean만 조회할 수 있는 기능만 있으면 되는데 그것보다 훨씬 무겁고 복잡한 Application Context를 주입받아야 되고 이에 따라 스프링 컨테이너에 종속적인 코드가 되버린다**는 점입니다. 참고로, getBean()을 호출하여 의존 관계를 주입 받는 것이 아니고 직접 **필요한 의존 관계를 찾는 것은 Dependency Injection(DI)가 아니고 Dependency Lookup(DL)**이라고 합니다.   
   
그건 그렇고 이 문제를 해결하기 위해 필요한 것을 다시 정리해보면, 저희는 사실 Application Context와 같이 복잡한 거 말고 DL 기능만 제공하는, 즉 **Spring Container에서 원하는 Bean만 찾을 수 있도록 제공하는 기능만 있으면 된다**는 겁니다. 이에 대해 **Spring은 친절하게 ObjectProvider<>와 ObjectFactory<>라는 것을 제공**합니다. 두 개의 차이점이라면, **ObjectFactory<>는 getObject()라는 기능만 제공하고 ObjectProvider<>는 그 이후에 나와 조금 더 편의 기능을 제공**합니다.   
    
ObjectProvider<>를 사용하는 방법을 코드를 통해 알아보겠습니다.   
    
```java
@Autowired
 private ObjectProvider<PrototypeBean> prototypeBeanProvider;
 public int logic() {
     PrototypeBean prototypeBean = prototypeBeanProvider.getObject();
     prototypeBean.addCount();
     int count = prototypeBean.getCount();
     return count;
}
```

코드는 이해하기 어렵지 않고 ObjectProvider<>로 조회할 PrototypeBean을 감싸서 의존 관계를 주입 받아 사용하면 됩니다. 이때 getObject() 메서드를 호출하는 시점에 직접 Spring Container에서 조회한다는 것만 알고 있으면 됩니다. getObject()가 호출될 때마다 조회가 되므로, Prototype Bean은 그때마다 새로 생성되게 됩니다.   
    
지금까지 알아본 **ObjectProvider<>와 ObjectFactory<> 모두 충분히 훌륭하고 좋은 방법이지만, 스프링 프레임워크에 의존적이라는 단점**이 있습니다. 이를 해결하고 싶다면 **JSR-330 자바 표준에 있는 Provider를 사용하면 됩니다.** 이 방법을 사용하기 위해서는 **javax.inject:javax.inject:1** 라이브러리를 gradle에 추가해줘야만 합니다.   
    
그렇게 라이브러리를 추가해주기만 하면 사용하는 방법은 크게 다르지 않습니다. 그냥 위 코드에서 **Provider<>로 바꿔주고 getObject()대신에 get()으로 바꿔주기만 하면 똑같이 동작**합니다.

## Web scope란?
이제부터는 앞에서 scope 종류 설명할 때 잠깐 언급되었던 Web과 관련된 scope들에 대해 자세히 알아보겠습니다. 이름 그대로 웹 환경에서만 동작하는 scope입니다. 그리고 이것들은 prototype scope랑 다르게 스프링이 종료시점까지 관리를 해줍니다. 따라서 종료 메서드가 호출됩니다.    
    
Web scope의 종류는 다음과 같습니다.   
    
1. request scope: HTTP 요청 하나가 들어오고 나갈 때까지 유지되는 스코프, 각각의 HTTP 요청마다 별도의 Bean 인스턴스가 생성되어 관리
2. session: HTTP session과 동일한 생명 주기를 가지는 스코프
3. application: 서블릿 컨텍스트(ServletContext)와 동일한 생명 주기를 가지는 스코프
4. websocket: 웹 소켓과 동일한 생명 주기를 가지는 스코프
   
![5](/assets/img/spring-bean-scope/5.png){: w="500" h="300" style="border:1px solid #eaeaea; border-radius: 7px; padding: 0px;"}

위 그림처럼 **request scope는 아무리 클라이언트 A,B가 동시에 요청을 보냈다고 해도 클라이언트 간에는 별도의 http request이기 때문에 전용 request scope Bean이 그때마다 생성**됩니다. 여기서 주의할 것은 **만약 Controller 외에도 Service 단까지 요청이 들어가서 request scope Bean을 요청한다면, 그때는 해당 http request에 따라 이미 생성된 request scope Bean을 그대로 사용**하게 됩니다. 이게 핵심인데, 이 **request scope를 활용하면 동시에 여러 HTTP 요청이 들어왔을 때 구분하기 쉽게 동일한 http request에 대해서 UUID나 request URL 등과 같은 정보들을 함께 logging하기가 쉬워집니다.**   
    
## 마무리
이번 포스팅에서는 지금까지 아무 말없이 넘어갔던 Spring Bean의 scope에 대해 알아봤습니다. Spring Bean scope 정의에서부터 다양한 종류 및 활용법까지 정리해봤는데, 개인적으로 request scope에 대해 공부하면서 평소에 관심있던 클라이언트 별 로깅 처리에 대한 맛보기를 함께 진행한 것 같아 뿌듯했습니다.   
    
이번 포스팅을 마지막으로 인프런 내 김영한 강사님의 스프링 핵심 원리 - 기본편 관련된 포스팅은 모두 마치도록 하겠습니다. 감사합니다.

## References
* 인프런 내 김영한 강사님의 스프링 핵심 원리 - 기본편